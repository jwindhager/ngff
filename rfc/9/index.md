# RFC-9: Zipped OME-Zarr

```{toctree}
:hidden:
:maxdepth: 1
reviews/index
comments/index
responses/index
versions/index
```

Add a specification for storing an OME-Zarr hierarchy within a ZIP archive.

## Status

This RFC is currently in state `R2` (waiting on reviewers).

| Name              | GitHub Handle | Institution                             | Date       | Status                                                          |
| ----------------- | ------------- | --------------------------------------- | ---------- | --------------------------------------------------------------- |
| Jonas Windhager   | @jwindhager   | SciLifeLab / Uppsala University, Sweden | 2025-07-02 | Corresponding Author [PR](https://github.com/ome/ngff/pull/316) |
| Norman Rzepka     | @normanrz     | scalable minds GmbH, Germany            | 2025-08-27 | Co-author [PR](https://github.com/ome/ngff/pull/316)            |
| Mark Kittisopikul | @mkitti       | HHMI Janelia, United States             | 2025-08-27 | Co-author [PR](https://github.com/ome/ngff/pull/316)            |
| Josh Moore        | @joshmoore    | German BioImaging e.V.                  | 2025-11-05 | Editor                                                          |

## Overview

The goal of this RFC is to standardize the storage of OME-Zarr hierarchies within ZIP archives as single files.

Specifically, this RFC aims to:

- **Improve user experience** by enabling the use of OME-Zarr in conventional file-based workflows, where reasonably small images stored on local desktop file systems are prevalent.
  This aims to ensure that a broad audience can benefit from existing and future features specific to OME-Zarr.
- **Simplify tool development** by offering a single-file storage for OME-Zarr that developers can easily integrate into file-centric applications.
  This aims to facilitate a broad adoption of OME-Zarr, even in tools that are not specifically made for large bioimaging datasets stored in the cloud.
- **Standardize existing practice** by formally specifying how to store OME-Zarr hierarchies in ZIP archives.
  This aims to facilitate interoperability among tools, to prevent suboptimal packaging of data, and to contribute to the standardization goals of the OME-NGFF community in general.

The RFC proposes to require the ZIP archive root to match the OME-Zarr root, prohibits ZIP-level compression, nested or multi-part ZIP archives and ZIP archive-level encryption, recommends performance optimizations (use of sharding codec, order of ZIP file entries), and defines a new file extension for OME-Zarr zip files.

## Background

OME-Zarr excels at storing large bioimaging datasets (often consisting of multiple images) in the cloud.
This is primarily achieved by storing individual image chunks as separate objects (object storage) or files on the file system (`DirectoryStore` implementation in Zarr v2, [file system store](https://zarr-specs.readthedocs.io/en/latest/v3/stores/filesystem/index.html) specification in Zarr v3).
However, for conventional use cases (e.g. reasonably small images stored on the local desktop file system), splitting a single image across multiple (few or many) files presents the following challenges:

**User experience-related challenges**:
Many tools in the bioimaging domain operate on individual files as independent entities (e.g. images).
For example, the "File open" dialog in ImageJ/Fiji lets users open single files as images.
Similarly, some operating systems expect an image to be stored in a single file, as apparent by e.g. file permission systems, file type concepts (e.g. file name extensions) and file type-dependent functionality (e.g., double/right-click, drag-and-drop, preview).
OME-Zarr, on the other hand, does not currently specify how to store data in a single file, but primarily relies on files distributed throughout nested directory structures.
As a consequence, the user experience of interacting with OME-Zarr data in conventional use cases lags behind "traditional" file formats such as (OME-)TIFF.
For example, users currently cannot associate an OME-Zarr file type with their favorite image viewer (no "double click" functionality), cannot effortlessly use their OME-Zarr images with existing file-centric tooling, nor can they easily share a few small OME-Zarr images with collaborators via file-oriented protocols (e.g. email attachments).
OME-Zarr's lack of support for file-centric workflows hampers its adoption in conventional use cases (and in turn the motivation for tool developers to support OME-Zarr).

**Challenges in tool development**:
Unlike most existing single-file formats, OME-Zarr organizes image data in hierarchical directory structures.
Since OME-Zarr represents images as directories, it is unclear to both users and developers how images should be be provided as input to software in otherwise file-centric environments:

- As a workaround, a `zarr.json` file could be provided as input.
  However, from a developer's perspective, it cannot be assumed that any given `zarr.json` file necessarily represents an image and its Zarr array(s) and groups without further validation.
  Furthermore, since file types are often determined based on file extensions and because the `.json` extension is not specific to OME-Zarr, developers would not be able to build upon file type-specific features/APIs of the operating system, making it more difficult to prioritize user experience.
- Instead of a single file, the directory representing the image itself could be provided as input.
  However, this would differ from other situations where users are expected to provide a single file as input, thus increasing overhead for implementing OME-Zarr support.
  Specifically, developers of otherwise file-centric tools would need to implement dedicated execution paths for handling directories, such as user interfaces for opening/saving data in/to directories, directory-specific drag-and-drop handlers, and advanced input validation logic.

In either case, the associated technical complexity may discourage tool developers from adopting OME-Zarr in a uniform and user-friendly fashion, which would in turn negatively affect user experience with OME-Zarr.

Taken together, these challenges unnecessarily differentiate OME-Zarr from other formats in use cases where single files may be appropriate.
Consequently, users and implementers have already begun to store OME-Zarr hierarchies in ZIP archives (see _Prior art and references_ section).
Because storage of OME-Zarr images in ZIP archives is not standardized, it is difficult to implement efficient storage and access patterns.
This can lead to format incompatibilities that may not only affect the adoption of OME-Zarr in conventional use cases, but may undermine the standardization goals of the OME-NGFF community as a whole.

## Proposal

To improve user experience with OME-Zarr in conventional use cases, standardize the storage of OME-Zarr hierarchies within ZIP archives.

Specifically:

- Add the ZIP format as a single-file storage container for OME-Zarr.
- Specify the location of OME-Zarr's root-level `zarr.json` within the ZIP archive.
- Recommend essential ZIP/Zarr storage parameters for creating OME-Zarr zip files.
- Disallow embedding OME-Zarr zip files in parent OME-Zarr hierarchies ("recursion").
- Define an OME-Zarr-specific file extension for OME-Zarr zip files: `.ozx`.

To minimize implementation effort and maximize compatibility, this RFC proposes a concrete archive file format as a single-file OME-Zarr storage container.

### The choice of ZIP
The ZIP archive file format was chosen for its simplicity, widespread adoption (e.g. library support in many programming languages, existing OME-Zarr implementations) and possibility for chunked file access enabled by its central directory, even on remote storage (see _Proposal_ section).

Note that the ZIP archive file format is already being used to realize comparable single-file formats in other domains, such as Java archives (.jar), Office Open XML (.docx, .pptx, .xlsx), or OpenDocument (.odt, .odt, .ods, .odg).

Since widespread adoption of single-file OME-Zarr across languages and tools is a primary goal of this RFC (see _Overview_), the choice of ZIP (over e.g. a bespoke binary format, see _Alternatives_ below) was weighed specifically against that goal:

- **Widespread, low-effort implementation support.** ZIP has near-ubiquitous library support across programming languages, so adding OME-Zarr zip file support is comparatively cheap for existing and new OME-Zarr implementations. Several implementations (e.g. zarr-python, tensorstore, zarrita.js) already had ZIP support before this RFC (see _Implementation_ section below), which this RFC takes as evidence that this low effort holds in practice.
- **Low effort scales from minimal to full-featured.** This RFC keeps the set of strict (MUST) requirements small (see _Specification_ section below), while most performance-related guidance is RECOMMENDED rather than required. A minimal, spec-compliant reader/writer is therefore cheap to build, and prototypes have shown that a fully recommendation-compliant writer is feasible with only moderate additional complexity, i.e. there is no large cliff between minimal and full-featured support.
- **Chunked access on local and remote stores alike.** The central directory enables efficient partial reads not just on local file systems, but also on HTTP(S), S3, and GCS object stores via range requests, matching how OME-Zarr is already accessed today.
- **OME-Zarr zip files are expected to be produced primarily by OME-Zarr-aware tooling**, not hand-zipped by end users with generic tools. The low implementation effort described above is intended to make it easy for such tools to adopt OME-Zarr zip file support, rather than to optimize for manual, generic-ZIP-tool-based workflows.

Unlike ZIP, a bespoke, purpose-built binary format would need to be implemented essentially from scratch, without existing libraries to build on, in every language and toolkit that wants to support single-file OME-Zarr.
This RFC weighs that additional per-implementation effort, and the resulting risk to widespread adoption, higher than the implementation-elegance benefits (e.g. a single, simpler reader/writer code path) that a bespoke format could offer.

Considering the intended use cases for zipped OME-Zarr, the advantages discussed above were considered to outweigh the disadvantages of the ZIP archive file format, such as its limitations in efficiently writing and accessing file contents (see _Drawbacks, risks, alternatives, and unknowns_ section below).

### OME-Zarr zip files as a file format

This RFC defines a file format for OME-Zarr: the OME-Zarr zip file, identified by the `.ozx` extension.
An OME-Zarr zip file is a valid ZIP file, and the contents obtained by unzipping it are a valid OME-Zarr (Zarr) hierarchy.
The format formalizes the storage of OME-Zarr within a ZIP file, with restrictions and recommendations that allow for good performance and for future extension through the archive comment.
It is a conservative subset of all possible ways of storing a Zarr hierarchy in a ZIP file and, because of these restrictions, a format of its own: every OME-Zarr zip file can be read by any ZIP tool and, once extracted, by any OME-Zarr tool, but not every ZIP file containing a Zarr hierarchy is an OME-Zarr zip file.

### Configuring ZIP for OME-Zarr
ZIP archives are traditionally associated with deflate compression which would have redundancy with the per-chunk compression existing in Zarr.
Changes in the size of files and compressed chunks could lead to significant fragmentation within a ZIP archive.

To enable the intended user experience (e.g. avoid additional prompting of users when opening OME-Zarr zip files), the location of the OME-Zarr root relative to the ZIP archive root needs to be specified.
In order to avoid inconsistencies when renaming OME-Zarr zip files, this RFC proposes to require the ZIP archive root to coincide with the OME-Zarr root directory.
In other words, according to this specification, the OME-Zarr's root-level `zarr.json` MUST be located in the root of the ZIP archive and not in a subfolder within the ZIP archive.
Potential problems (e.g. loss of data) resulting from "accidentally" extracting the ZIP archive in-place (e.g. using on-board tooling of some operating systems) can be alleviated by introducing a custom file extension (see below).

To facilitate efficient storage and access of OME-Zarr zip files, a set of essential ZIP/Zarr parameters are specified or recommended in this RFC.
ZIP-level compression is prohibited: all entries MUST use the STORE method (no compression).
This avoids unnecessary compression of already compressed data (e.g. when using Zarr compression codecs) and makes it easier to directly conduct partial reads of the ZIP archive.
This restriction may be relaxed in the future, e.g. for `zarr.json` documents.
The following are recommended:

- Use the ZIP64 format.
  The ZIP format itself requires ZIP64 once an archive contains 65,535 or more entries, or once an archive, a central directory, an entry or the offset of an entry reaches 4 GiB.
  This RFC recommends that the ZIP64 end of central directory records are present even when they are not yet required, which costs about 76 bytes once per archive.
  This makes it easier to append to an OME-Zarr zip file as it passes through the thresholds on the number of entries and on the size and offset of the central directory, because the archive does not change format along the way.
  Entries that are or may become larger than 4 GiB, which is typical for shards, are recommended to use the ZIP64 extra fields, which may be needed already when the entry is written if its final size is not known in advance.
  Writers may use the ZIP64 extra fields for all entries; this adds tens of bytes per entry (an extra field of 20 bytes in each local file header and, depending on the library, up to about 32 bytes in each central directory record), which is small relative to the expected sizes of OME-Zarr datasets of gigabytes to terabytes, especially when the sharding codec keeps the number of entries small.
  These are recommendations rather than requirements because not all ZIP libraries currently offer an option to write ZIP64 for small archives; OME-Zarr zip files that do not use ZIP64 remain valid as long as they do not exceed the limits of the classic ZIP format.
- Use the Zarr sharding codec.
  This reduces the number of records in the central directory.
- Place all `zarr.json` records at the beginning of the central directory, so that every `zarr.json` precedes every other record, except for directory entries.
  This enables efficient metadata processing and discovery of the hierarchy structure without parsing the entire central directory.
  Only the order of the records in the central directory matters for this purpose; the central directory is rewritten whenever the archive is appended to, so it can be put in this order when the archive is closed, whatever the order in which the entries themselves were written.
- Include an OME-Zarr-specific archive comment in the ZIP file header, indicating compliance with the OME-Zarr specification.
  This further facilitates efficient data/metadata access and also allows for additional (optional/recommended) single-file metadata that may be specified in future OME-Zarr versions.

This RFC explicitly prohibits embedding an OME-Zarr zip file as subhierarchy of a parent OME-Zarr hierarchy.
In particular this prohibits "recursive zipping", the embedding of an OME-Zarr zip file within a parent OME-Zarr zip file.

Furthermore, this RFC prohibits splitting up the ZIP archive into multiple files ("multi-volume archives"), in favor of directory-backed OME-Zarr and Zarr's sharding codec.

Likewise, this RFC prohibits encryption at the ZIP archive level, since readers are not expected to handle passwords or decryption and an encrypted archive could not be opened by generic OME-Zarr tooling.
Encryption at the codec level may be applicable to Zarr in general, but is out of scope for this RFC.

#### Access and mutability

Streaming an OME-Zarr zip file from beginning to end is not a goal of this RFC.
As with the rest of Zarr, remote access is intended to use range-read requests, which are available from standard HTTP servers and object storage services such as S3.
Client software is expected to retrieve the central directory, located near the end of the archive, without reading the entire archive, and then read the individual entries it needs.

Some mutability of OME-Zarr zip files is expected.
Files can be appended to the archive and file contents can be modified in-place, and the central directory can be rewritten to omit obsolete data or files.
The central directory may contain records with the same name transiently while an archive is being appended to; writers SHOULD remove such duplicates from the central directory at the end of a writing session.
Recommending ZIP64 (see above) is meant to support this, since it allows an archive to keep growing beyond the limits of the classic ZIP format.
Mutation is, however, not efficient for every workload (see _Drawbacks, risks, alternatives, and unknowns_ below).
For massively parallel changes, it is expected that the Zarr arrays are extracted from the archive, modified, and then repacked.
An unzipped OME-Zarr may provide better write performance, especially when writes are parallelized.
The recommendations of this RFC allow for some updates of the data in an OME-Zarr zip file while acknowledging this limitation.

Finally, this RFC also defines a new file extension to be used specifically with OME-Zarr zip files.
This should enable file type detection (in absence of a magic number), improve user experience (e.g. by enabling file type association), avoid "accidental" in-place extraction (e.g. using on-board tooling of some operating systems) and encourage the use of OME-Zarr-specific tooling for creating OME-Zarr zip files (to follow the recommendations listed earlier).

Note that this RFC does not - semantically or otherwise - restrict the data content of OME-Zarr hierarchies to be stored in OME-Zarr zip files.

### Intended use cases and limitations

The OME-Zarr zip file is primarily intended as a format that is written once and read many times, for the following use cases:

- **File-centric workflows on a desktop**: opening an image from a file dialog, associating the file type with a viewer ("double click"), drag and drop, and sharing a few small images as e-mail attachments (see _Background_).
- **Transport and archival** of OME-Zarr datasets as a single object, including large ones. The discussion of this RFC reported terabyte-scale Zarr hierarchies that are already distributed as zip, tar or squashfs files for this reason ([comment](https://github.com/ome/ngff/pull/316#issuecomment-3214762172)).
- **Viewing and exploring remote data without downloading it**, using HTTP range requests. This is demonstrated by the viewers and example files listed in the _Implementation_ section: Neuroglancer, WEBKNOSSOS (which can read an OME-Zarr zip file remotely) and Zipglancer (which lists the contents and structure of OME-Zarr zip files using range requests).
- **Exchange between Zarr implementations**, several of which already support ZIP stores (see _Implementation_).

OME-Zarr zip files are not intended for, and are not appropriate for:

- workloads with many concurrent writers or frequent in-place modification, for which it is expected that the Zarr arrays are extracted, modified and repacked, or that an unzipped OME-Zarr is used (see _Access and mutability_);
- streaming from beginning to end (see _Access and mutability_);
- archive-level encryption (see _Specification_);
- very large numbers of entries without the sharding codec (see _Performance_).

## Specification

Amend the specification with the following new section, which is reproduced in full below:

### Single-file OME-Zarr

This section specifies how to store an OME-Zarr hierarchy within a single file.

#### OME-Zarr zip files

An OME-Zarr hierarchy MAY be stored within a ZIP archive.

For a ZIP file to be referred to as an OME-Zarr zip file the following conditions MUST be met:

1. The ZIP file MUST contain exactly one OME-Zarr hierarchy.
2. The root of the ZIP archive MUST correspond to the root of the OME-Zarr hierarchy. The ZIP file MUST contain the OME-Zarr's root-level `zarr.json`.
3. OME-Zarr zip files MUST NOT be embedded in a parent OME-Zarr hierarchy (as a sub-hierarchy or otherwise).
4. OME-Zarr zip files MUST NOT be split into multiple parts.
5. OME-Zarr zip files MUST NOT use ZIP archive-level encryption.
6. ZIP-level compression MUST NOT be used: all ZIP entries MUST use the STORE method. Compression is expected to be performed by Zarr-level codecs.
7. ZIP entries MUST NOT contain extra fields (in the local file header or the central directory) other than the following standard metadata fields:
   - ZIP64 extended information (header ID `0x0001`),
   - Extended timestamp (header ID `0x5455`),
   - Info-ZIP New Unix (header ID `0x7875`).

   This list may be extended in future versions of this specification.
8. OME-Zarr zip files MUST NOT contain data before the first local file header, i.e. the first local file header MUST be at offset 0 of the file. In particular, this prohibits self-extracting ZIP archives and other prepended executable stubs. This may be revisited in a future version of this specification.

OME-Zarr zip files MAY contain directory entries, i.e. zero-length entries whose names end with `/`.
Directory entries are not Zarr keys and SHOULD be ignored by readers.
Directory entries are ignored by the `jsonFirst` parameter described below: they may appear anywhere in the central directory, including before the `zarr.json` records.

A validator for OME-Zarr zip files MUST report a violation of any of the MUST or MUST NOT requirements above.

When creating OME-Zarr zip files, the following are RECOMMENDED. They are intended to ensure that reading OME-Zarr zip files is similarly performant as reading from other storage formats (see the _Performance_ section):

1. The ZIP64 end of central directory records SHOULD be present, irrespective of the size of the archive.
2. Entries that are or may become larger than 4 GiB SHOULD use the ZIP64 extra fields. Writers MAY use them for all entries.
3. The sharding codec SHOULD be used to reduce the number of entries within the ZIP archive, depending on the chunk size, the expected number of chunks and the codec pipeline (see _Drawbacks_).
4. All `zarr.json` records SHOULD precede all other records in the central directory, not counting directory entries. In this case, the `jsonFirst` parameter of the archive comment SHOULD be set to `true` (see below). Within the `zarr.json` records, the root-level `zarr.json` SHOULD come first and the other `zarr.json` records SHOULD follow in breadth-first order. The order in which the entries themselves are stored in the archive is not restricted.
5. The name of OME-Zarr zip files SHOULD end with `.ozx`.
6. The ZIP archive comment SHOULD contain an UTF-8-encoded JSON string with an `ome` attribute that holds a `version` key with the OME-Zarr version as string value, such that `{"ome": { "version": "XX.YY" }}` is the minimum recommended content. Additional optional content is described in the next section.

#### OME-Zarr Zip Comment Structure

The zip comment is intended to provide metadata pertinent to the zip file structure, such as information about the ordering of entries within the central directory. It is not intended for storing metadata about the OME-Zarr's content. Such content-related metadata should be stored within the OME-Zarr hierarchy.

The zip comment is encoded as JSON so that future versions of the OME-Zarr specification can add parameters to it, enabling parameterized features beyond versioning.
The keys under the top-level `ome` attribute are strictly defined by this specification and are extended only by future versions of the specification.
Other top-level keys of the JSON object are permitted and are meant to allow composition with other specifications.

The `ome` attribute in the zip archive comment MAY contain a `zipFile` attribute, which in turn MAY contain a `centralDirectory` attribute. The `centralDirectory` attribute provides metadata about the central directory's structure and content.

The `centralDirectory` attribute MAY contain the following key:

- `jsonFirst`: If `true`, this asserts that every `zarr.json` record precedes every other record in the central directory, not counting directory entries, which may appear anywhere. It does not assert any particular order among the `zarr.json` records, such as breadth-first order. This allows the hierarchical structure of the contents to be discovered without parsing the entire central directory, which could contain many records of Zarr chunks: a reader can stop parsing at the first record that is neither a `zarr.json` nor a directory entry. Implementations MAY assume that no further `zarr.json` records exist beyond the first record that is neither a `zarr.json` nor a directory entry if `jsonFirst` is `true`. If `jsonFirst` is omitted, the value defaults to `false`.
  The intended use is to let viewers of OME-Zarr zip files, similar to tree views in HDF5 viewers such as HDFView or h5web, quickly display the structure of the hierarchy before reading any array data, and to support features such as auto-completion.
  Without `jsonFirst` set to `true`, a reader has to parse the entire central directory to be sure that the whole structure has been discovered.
  `jsonFirst` is a parameter because `false` is a valid value: files that do not order their entries this way remain valid OME-Zarr zip files, although ordering is recommended for the use cases above.
  Since readers may rely on `jsonFirst: true` to stop reading the central directory early, a writer that modifies an archive SHOULD ensure that the flags in the archive comment are consistent with the order of the central directory, either by restoring the order or by setting `jsonFirst` to `false` (or omitting it).

For example,
```json
{
  "ome": {
    "version": "XX.YY",
    "zipFile": {
      "centralDirectory": {
        "jsonFirst": true
      }
    }
  }
}
```

For example, for a hierarchy with the following `zarr.json` files:
```
/
├── zarr.json
├── image/
│   ├── zarr.json
│   ├── s0/
│   │   └── zarr.json
│   └── s1/
│       └── zarr.json
└── labels/
    └── zarr.json
```
a central directory with `jsonFirst: true` and, in addition, the recommended breadth-first order of the `zarr.json` records is:
1. `zarr.json`
2. `image/zarr.json`
3. `labels/zarr.json`
4. `image/s0/zarr.json`
5. `image/s1/zarr.json`
6. (followed by the records of all other entries, such as chunks and shards)

An order such as `zarr.json`, `image/zarr.json`, `image/s0/zarr.json`, `image/s1/zarr.json`, `labels/zarr.json` (depth-first) is also compatible with `jsonFirst: true`, because only the precedence of all `zarr.json` records over all other records is asserted.

## Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [IETF RFC 2119](https://tools.ietf.org/html/rfc2119).

## Stakeholders

In principle:

- People who work in conventional settings (e.g. reasonably small images stored on the local file system) and want to use OME-Zarr
- Developers who want to make their new or existing tools available to both conventional use cases and use cases poised for OME-Zarr
- Anyone benefitting from further file format standardization/OME-Zarr adoption within the bioimaging community

The storage of single images as single-file (e.g. zipped) OME-Zarrs has been frequently requested in online forums, community calls, events, GitHub issues, etc.
While too numerous to list here, relevant search phrases include "OME-Zarr single file", "OME-Zarr zip", "OME-Zarr local file system" and "Zarr ZipStore".

Facilitator: Josh Moore (German BioImaging)

Suggested reviewers:

- BioImage Archive team members (EMBL-EBI, United Kingdom), e.g. @matthewh-ebi @kbab
- Curtis Rueden (University of Wisconsin-Madison, United States) @ctrueden
- Lenard Spiecker (Miltenyi, Germany)
- Luca Marconato (EMBL Heidelberg, Germany) @LucaMarconato
- Pete Bankhead (University of Edinburgh, United Kingdom) @petebankhead
- Tong Li (Wellcome Sanger Institute, United Kingdom) @BioinfoTongLI

Consulted: everyone mentioned in [PR #316](https://github.com/ome/ngff/pull/316)

Socialization: see Prior art and references; the draft was further discussed among co-authors on August 27th, 2025 ([minutes](https://hackmd.io/@eAPtiynRTLOLwk5Vzg8AqQ/rJ403lIYlg)).

## Implementation

### Converters, validators
- A first implementation has been [prototyped](https://github.com/ome/ngff/pull/316#issuecomment-3302456557) by one of the coauthors.
- [ozx-tck](https://github.com/clbarnes/ozx-tck) is a toolkit to validate existing .ozx files and generate valid, warning, and error test cases.
- [ngff-zarr](https://ngff-zarr.readthedocs.io/en/latest/) is a Python-based toolkit for working with .ozx files.

### Viewers
- [Neuroglancer](https://neuroglancer-demo.appspot.com/#!%7B%22dimensions%22:%7B%22x%22:%5B3.6039815346402084e-7%2C%22m%22%5D%2C%22y%22:%5B3.6039815346402084e-7%2C%22m%22%5D%2C%22z%22:%5B5.002025531914894e-7%2C%22m%22%5D%7D%2C%22position%22:%5B135%2C137%2C118%5D%2C%22crossSectionScale%22:1%2C%22projectionScale%22:512%2C%22layers%22:%5B%7B%22type%22:%22image%22%2C%22source%22:%22https://static.webknossos.org/misc/6001240.ozx%7Czip:%7Czarr3:%22%2C%22localDimensions%22:%7B%22c%27%22:%5B1%2C%22%22%5D%7D%2C%22localPosition%22:%5B0%5D%2C%22tab%22:%22source%22%2C%22opacity%22:1%2C%22blend%22:%22additive%22%2C%22shader%22:%22#uicontrol%20invlerp%20contrast%5Cn#uicontrol%20vec3%20color%20color%5Cnvoid%20main%28%29%20%7B%5Cn%20%20float%20contrast_value%20=%20contrast%28%29%3B%5Cn%20%20if%20%28VOLUME_RENDERING%29%20%7B%5Cn%20%20%20%20emitRGBA%28vec4%28color%20%2A%20contrast_value%2C%20contrast_value%29%29%3B%5Cn%20%20%7D%5Cn%20%20else%20%7B%5Cn%20%20%20%20emitRGB%28color%20%2A%20contrast_value%29%3B%5Cn%20%20%7D%5Cn%7D%5Cn%22%2C%22shaderControls%22:%7B%22contrast%22:%7B%22range%22:%5B7%2C927%5D%2C%22window%22:%5B0%2C1159%5D%7D%2C%22color%22:%22#ff0000%22%7D%2C%22volumeRenderingDepthSamples%22:256%2C%22name%22:%226001240.ozx%20c-0.5%22%7D%2C%7B%22type%22:%22image%22%2C%22source%22:%22https://static.webknossos.org/misc/6001240.ozx%7Czip:%7Czarr3:%22%2C%22localDimensions%22:%7B%22c%27%22:%5B1%2C%22%22%5D%7D%2C%22localPosition%22:%5B1%5D%2C%22tab%22:%22source%22%2C%22opacity%22:1%2C%22blend%22:%22additive%22%2C%22shader%22:%22#uicontrol%20invlerp%20contrast%5Cn#uicontrol%20vec3%20color%20color%5Cnvoid%20main%28%29%20%7B%5Cn%20%20float%20contrast_value%20=%20contrast%28%29%3B%5Cn%20%20if%20%28VOLUME_RENDERING%29%20%7B%5Cn%20%20%20%20emitRGBA%28vec4%28color%20%2A%20contrast_value%2C%20contrast_value%29%29%3B%5Cn%20%20%7D%5Cn%20%20else%20%7B%5Cn%20%20%20%20emitRGB%28color%20%2A%20contrast_value%29%3B%5Cn%20%20%7D%5Cn%7D%5Cn%22%2C%22shaderControls%22:%7B%22contrast%22:%7B%22range%22:%5B25%2C824%5D%2C%22window%22:%5B0%2C1025%5D%7D%2C%22color%22:%22#00ff00%22%7D%2C%22volumeRenderingDepthSamples%22:256%2C%22name%22:%226001240.ozx%20c0.5%22%7D%5D%2C%22selectedLayer%22:%7B%22visible%22:true%2C%22layer%22:%226001240.ozx%20c-0.5%22%7D%2C%22layout%22:%224panel-alt%22%2C%22helpPanel%22:%7B%22row%22:2%7D%2C%22settingsPanel%22:%7B%22row%22:3%7D%2C%22toolPalettes%22:%7B%22Shader%20controls%22:%7B%22side%22:%22left%22%2C%22row%22:1%2C%22query%22:%22type:shaderControl%22%7D%7D%7D) of the [generated data](https://static.webknossos.org/misc/6001240.ozx) has kindly been [made available](https://github.com/ome/ngff/pull/316#issuecomment-3302595684) by Davis Bennett.
- [WEBKNOSSOS](https://github.com/scalableminds/webknossos/pull/9738) is a web-based viewing and annotation platform that supports .ozx files.
- [Zipglancer](https://github.com/JaneliaSciComp/zipglancer) is a client-side web explorer for ZIP and .ozx archives, using HTTP range requests, that reads the `jsonFirst` setting from the archive comment.

### Example datasets
- [6001240.ozx](https://static.webknossos.org/misc/6001240.ozx) is the dataset shown in the Neuroglancer link above.
- 51 example datasets (CT, MRI and simulation volumes) are provided as OME-Zarr zip files in a public bucket, `https://ome-zarr-scivis.s3.us-east-1.amazonaws.com/v0.5/96x2-ozx/<name>.ozx`, listed in [Zipglancer](https://github.com/JaneliaSciComp/zipglancer/blob/main/src/data/sciVisDatasets.ts).

### Zarr libraries
- [zarr-python](https://github.com/zarr-developers/zarr-python) has a ZipStore.
- [zarr-java](https://github.com/zarr-developers/zarr-java) has OME-Zarr metadata support and a ZipStore, which adheres to the RFC-9 specification.
- [zarrita.js](https://github.com/manzt/zarrita.js) is a JavaScript library for reading and writing OME-Zarr files, including .ozx files.
- [zarrs](https://github.com/zarrs/zarrs_zip) has a ZipStore and a [converter for .ozx](https://github.com/clbarnes/ozx).
- [tensorstore](https://google.github.io/tensorstore/kvstore/zip/index.html) has a ZipFileStore.

Note for implementers: Python's `zipfile` permits several entries with the same name ([cpython issue 47073](https://github.com/python/cpython/issues/47073)), so rewriting a `zarr.json` produces duplicate central directory records, which this RFC recommends to remove at the end of a writing session. Data and metadata of a hierarchy are conceptually written together and a library has to keep the central directory consistent with both.



## Drawbacks, risks, alternatives, and unknowns

Drawbacks:

- **Disadvantages of the ZIP archive file format** when writing and accessing file contents relate to the structure of the central directory.
  - **The zip file central directory is a flat, non-hierarchical list near the end of the file**.
    While zip files contain a compact central directory that other archive file formats lack, the directory is a simple list occurring in no particular order.
    This specification exploits the lack of order to list the zarr.json files first, consolidating the metadata while outlining the Zarr hierarchy.
    Implementations may need to parse the directory into a hash table structure or sort the directory to facilitate locating files efficiently.
  - **A large number of entries may make the central directory difficult to parse or search directly**.
    The time complexity of the reading the directory is `O(N)` where `N` is the number of all files in the archive.
    A single array containing a large number of chunks in individual files may adversely affect the ability to locate chunks in other arrays contained within the archive.
    To mitigate this, shards are recommended to decrease the number of entries in the central directory.
    The use of the `sharding_indexed` codec delegates the indexing of the numerous chunks to the ordered index in the shard.
    With Zarr shards, the zip central directory may then mainly consist of entries describing the location of array and group metadata as well as the location of the shards.
  - **Adding files to the zip file requires rewriting the central directory**.
    Since the central directory occurs near the end of the file, adding new files or expanding existing files requires that the central directory be removed, the new content added, and then central directory rewritten.
    This can be mitigated by adding many files at once and then closing the file once rather than adding them one by one and closing the file after each addition.
  - **File content is not necessarily page-aligned**.
    Compared to a directory store, the content of a file within a ZIP archive does not generally start at a page boundary of the storage device.
    Implementers using unbuffered, page-aligned I/O reported a significant performance impact for both reading and writing, due to read-modify-write cycles.
    This can be avoided by allocating a separate page for each local file header, including for chunks inside shards and the shard index, and leaving partially filled pages empty.
    This comes at the cost of additional space, which is acceptable when sharding is used and chunks are not very small.
  - **Individual file headers may describe obsolete files**.
    While the central directory contains the canonical list of files near the end of a zip archive, obsolete files and their file headers may still be present earlier in the archive.
    Files detected while streaming a zip file may not represent the latest version of a file or files that may have been deleted.
    It may be advantageous to extract individual files to manipulate them and then rebuild the archive when processing is complete rather than trying to modify files within the archive.
  - **Removing a file from a zip archive, may not reduce the file size of an archive depending on the implementation**.
    Removing a file from a zip archive may only remove the file's entry in the central directory.
    Free space within a zip archive is not explicitly tracked and thus cannot be easily reclaimed or reused.
    Therefore, it is not recommended to overwrite or delete files within a zip archive frequently such as during image processing operations.
  - **Sharding constrains partial writes**.
    Partial writes of a shard are impractical unless the final size of each shard is known in advance, so it is often not recommended to shard an axis that is acquired sequentially (e.g. a Z-axis acquired slice by slice cannot be written chunk by chunk when sharded along Z, unless the codec pipeline produces a fixed size or a whole slice constitutes a single shard).
    The recommendation to use sharding therefore depends on the chunk size, the expected number of chunks and the codec pipeline.
  - **ZIP requires a CRC-32 for every entry**, including entries stored without compression, which is useful for integrity verification but burdens partial writes, appends and partial reads: the CRC-32 covers the whole entry, so it has to be recomputed when a shard is modified.
    Implementations may validate at the granularity of shards or chunks and defer CRC-32 checks for entries that are still being written.
    It has also been reported that x86_64 processors provide hardware support for CRC-32C (SSE4.2) but not for the CRC-32 used by ZIP, so computing it can be comparatively slow.
  - **Updating an entry in place is generally not supported by ZIP libraries**.
    Some implementations reserve capacity (padding) for metadata so that a small update does not require appending a new copy of the entry and rewriting the central directory, similar to `tiffcomment` or `tiffset` for TIFF files.
    Padding by means of ZIP extra fields is not permitted by this RFC (see _Specification_); reserving unused space between entries remains possible.
  - **The size of a single file is bounded**, in principle by the 64-bit size limits of ZIP64 (which are far beyond practical sizes) and in practice by the handling of very large single files: file system and object store limits (for example the maximum size of a single object), and the cost of transferring or copying one very large file.
    For very large datasets, a directory-backed or object store-backed OME-Zarr remains an option.

These disadvantages were considered to be outweighed by other aspects (see _Proposal_ section).

Risks:

- **Existing OME-Zarr implementations may require adaptation**.
  However, adaptation effort is expected to be manageable, as many existing implementations already support zipped OME-Zarr.
- **Developers may choose to only support single-file OME-Zarr** (partial OME-Zarr support).
  However, most software relies on third-party packages for reading/writing OME-Zarr, which implement the full OME-Zarr stack.
- **Users may employ OME-Zarr-agnostic tooling to "zip" OME-Zarr**, resulting in non-compliant or suboptimally stored zipped OME-Zarr.
  This risk is mitigated by introducing a custom file extension for OME-Zarr zip files, requiring manual - and thus deliberate - renaming of ZIP archives created using generic tooling, akin to similar file formats (see _Prior art and references_ section).

Alternatives:

- **Do not specify a single-file variant** of OME-Zarr.
  Drawbacks of this alternative were discussed extensively in the _Background_ section of this RFC.
- **Use HDF5 or a similar generic single-file container format** as storage backend instead of Zarr.
  However, creating a "completely new" file format (e.g. "OME-HDF5"; as opposed to building upon OME-Zarr) would harm the standardization efforts of the OME-NGFF community. Additionally, library support for HDF5 is currently not optimized for remote storage, which is a requirement for the use cases of OME-Zarr.
- **Use TIFF as storage backend** instead of Zarr, e.g. with the `zarr.json` contents embedded in the `ImageDescription` tag, and optionally appended with a Zarr shard index.
  However, this would similarly harm aforementioned standardization efforts and would further restrict file contents to single volumes.
- **Use an archive file format other than ZIP**.
  Among other reasons, the ZIP format was chosen for its widespread adoption and support for chunked file access (see _Proposal_ section).
  Other widely used formats, such as TAR, could possibly be adapted to enable chunked file access, but the gained advantages over ZIP were not considered to outweigh the required specification complexity and additional implementation effort. In particular, TAR does not have a central directory, which is required for efficient random access to file contents.
- **Use a custom, purpose-built binary format** instead of an existing container format.
  Such a format could minimize complexity by embedding structural (e.g. chunk/shard offset) metadata up front, tailored specifically to OME-Zarr, and would only require a single reader/writer code path, avoiding some of the disadvantages of the ZIP format discussed above.
  However, unlike ZIP, it would need to be implemented essentially from scratch, without existing libraries to build on, in every language and toolkit intending to support single-file OME-Zarr.
  This RFC weighs the resulting cost to widespread adoption higher than the implementation-elegance gained this way (see _Proposal_ section).
- **Address the single-file issue on the Zarr-level**, e.g. by adding a ZipStore to the Zarr v3 specification.
  However, this likely would not cover all aspects proposed in this PR (e.g. file extension, ZIP restrictions) and it is unclear if and when ongoing efforts in this direction will be successful.
  If a ZipStore is added to the Zarr specification after acceptance of this RFC, the OME-Zarr specification can be amended as necessary at a later stage.

Unknowns:

- **This RFC may inspire further specialization** ("profiles") of OME-Zarr in the future.

## Abandoned Ideas

The following ideas were abandoned:

- **Limit single-file OME-Zarr to single volumes**.
  Such a specialization would be programmatically verifyable and would bring OME-Zarr closer to existing single-volume-per-file image formats.
  However, it would simultaneously restrict single-file OME-Zarr to use cases that are already well-served by existing single-file formats, sacrificing features unique to OME-Zarr (e.g. coordinate systems/transforms, collections, derived data) that form part of the original motivation for this RFC.
  _Note that these drawbacks could potentially be addressed in the future by allowing the embedding of single-file OME-Zarr in larger, more complex OME-Zarr hierarchies, or by the collections RFC._
- **Semantically restrict the contents** of single-file OME-Zarr.
  The underlying idea of such a specialization would be to specify a common set of interaction patterns that is shared among different software.
  For example, one could attempt to restrict the contents of a single-file OME-Zarr to capabilities shared among image viewers, so that these programs e.g. do not need to show additional prompts for which parts of the data to load and display.
  However, such a semantic specification would necessarily depend on the capabilities of the software considered and would therefore be inherently incomplete as well as difficult to formulate in a generic, software-agnostic fashion.
  Moreover, this idea is orthogonal to that of a zipped single-file format: one could equally propose semantic restrictions on multi-file OME-Zarr, which could have value, but such a discussion is outside the scope of this RFC.
  Furthermore, a semantic specification would likely not be programmatically verifyable.
- **Do not specify a concrete storage backend** for single-file OME-Zarr.
  Initial drafts of this RFC attempted to specify an abstract, backend-agnostic single-file OME-Zarr format.
  However, this would allow for an infinite number of concrete realizations (e.g. zip, tar, ...), thereby undermining the standardization efforts of the OME-NGFF community.
- **Specify ZIP as the one and only storage backend** for single-file OME-Zarr.
  This would unnecessarily limit the space for future innovation/specialization in the OME-Zarr specification.
- **Use a file extension other than `.ozx`**.
  The following candidates were considered:
  - `.zarrx` or `.zar` - not OME-specific <!-- codespell:ignore -->
  - Multi-part file extensions (e.g. `.ome.zarr.zip`, `.ome.zarrx`, `.ome.zar`) - suboptimal user experience <!-- codespell:ignore -->
  - Any other permutation of `oz[pzx]` that is not yet in active use by other software

## Prior art and references

Prior discussions related to (OME-)Zarr specifications:

- [zarr-developers/zarr-specs#311](https://github.com/zarr-developers/zarr-specs/pull/311): _Draft zip file store specification_
- [2024 OME-NGFF workflows hackathon](https://doi.org/10.37044/osf.io/5uhwz_v2) (Lüthi et al., 2025), Section 2.2.3: _Single-file ("ZIP") OME-Zarr_
- The [zipstorers](https://ossci.zulipchat.com/#narrow/channel/423692-Zarr/topic/.E2.9C.94.20zipstorers/with/527178266) topic in the Open Source Science (OSSci) Initiative Zulip chat
- The [Single File Format Detection](https://imagesc.zulipchat.com/#narrow/channel/212929-general/topic/Single.20File.20Format.20Detection/with/536692137) topic in the image.sc Zulip chat
- The [.zarr.zip on s3](https://github.com/zarr-developers/zarr-python/discussions/1613) discussion on the zarr-python GitHub repository

Existing single-file (zipped) OME-Zarr implementations:

- [2024 OME-NGFF workflows hackathon](https://doi.org/10.37044/osf.io/5uhwz_v2) (Lüthi et al., 2025), Table 1: _Surveyed Zarr implementations and their capabilities to read single archive files_

Existing single-file (zipped) OME-Zarr datasets:

- [Open OME-Zarr image datasets with "ZIP" file type on Zenodo](https://zenodo.org/search?q=OME-Zarr&f=resource_type%3Adataset&f=access_status%3Aopen&f=resource_type%3Aimage&f=file_type%3Azip&l=list&p=1&s=10&sort=bestmatch)
- [SpatialData spatial omics example datasets](https://spatialdata.scverse.org/en/latest/tutorials/notebooks/datasets/README.html#spatial-omics-datasets)

Related concepts and file formats:

- Java archives (.jar)
- Office Open XML (.docx, .pptx, .xlsx)
- OpenDocument (.odt, .odp, .ods, .odg)
- OmniGraffle documents (.graffle)
- Blender's [packed data](https://docs.blender.org/manual/en/latest/files/blend/packed_data.html)

The European Space Agency (ESA) has decided to disseminate Sentinel-2 satellite images as zipped Zarr, see the [Earth Observation Platform framework documentation](https://cpm.pages.eopf.copernicus.eu/eopf-cpm/main/PSFD/4-storage-formats.html).
Data can for example be obtained as zipped Zarr from the [EOPF Sentinel Zarr Samples Service STAC API](https://stac.browser.user.eopf.eodc.eu/).

## Future possibilities

In the future, the following could be considered:

- Reduce the OME-Zarr zip specification to aspects not covered by the Zarr ZipStore specification (once available)
- Allow embedding of OME-Zarr zip files in parent OME-Zarr hierarchies
- Allow embedding of OME-Zarr zip files in parent OME-Zarr zip files
- Specify a single-volume specialization of OME-Zarr zip files
- Allow ZIP-level compression of `zarr.json` documents
- Allow the root of the OME-Zarr hierarchy to be located at a configurable path within the archive, instead of at the root of the ZIP archive

## Performance

Performance of reading OME-Zarr zip files is expected to be comparable to reading other Zarr stores, provided that readers use range requests to read the central directory and the requested entries rather than reading the entire archive, and that the recommendations in the _Specification_ section are followed (in particular the use of sharding to limit the number of central directory entries).
Fast random access to chunks, one of the principal benefits of Zarr, is retained because ZIP entries are stored without compression and can be located through the central directory.

This expectation is supported by the following external evaluations, whose most relevant findings are summarized here:

- Olli Niemitalo and Otto Rosenberg (Häme University of Applied Sciences, Finland) [evaluated](https://github.com/hamk-uas/datacube-storage-lab) the performance of using zipped Zarr files for training machine learning models on geospatial data (Sentinel 2 Level-1C; tiled raster images). On an S3 object store, reading zipped Zarr with an asynchronous filesystem implementation took about 7.1 s, compared to about 7.7 s for the equivalent directory Zarr; on local NVMe storage the times were about 1.3 s (zipped) and 1.2 s (directory). In a separate run, the standard synchronous ZipStore implementation was substantially slower than directory Zarr (about 24 s versus about 7.5 s on S3, and about 3.2 s versus about 1.1 s on NVMe), i.e. the reader implementation matters more than the use of ZIP itself. Copying the zipped dataset as a single file also took about 10 to 12 minutes, compared to about 27 to 30 minutes for the corresponding directory of 8,562 files.
- The decision of the European Space Agency to disseminate Sentinel-2 images as zipped Zarr was accompanied by a [report](https://github.com/csaybar/ESA-zar-zip-decision/issues/6) that an asynchronous filesystem implementation reading from zip files can read zipped Zarr no slower than directory Zarr.

These evaluations concern geospatial raster data, not microscopy data, and were not performed by the authors of this RFC.
Writers should verify that reading their OME-Zarr zip files is similarly performant as reading from other storage formats for their data.

## Compatibility

This proposal adds a new feature to the OME-Zarr specification.
As such, it is fully backwards-compatible, but not forwards-compatible.
Implementations are expected to adopt the added support for OME-Zarr zip files.
This RFC applies only to OME-Zarr v0.5 and later, which dependend on Zarr version 3.

## Testing

Testing will involve the creation of public example data and the adaptation of validators.
The generated example OME-Zarr zip file can then be used to test existing implementations.

## Tutorials and Examples

A first [example dataset](https://static.webknossos.org/misc/6001240.ozx) has been [created](https://github.com/ome/ngff/pull/316#issuecomment-3302456557) by one of the coauthors.
A [neuroglancer view](https://neuroglancer-demo.appspot.com/#!%7B%22dimensions%22:%7B%22x%22:%5B3.6039815346402084e-7%2C%22m%22%5D%2C%22y%22:%5B3.6039815346402084e-7%2C%22m%22%5D%2C%22z%22:%5B5.002025531914894e-7%2C%22m%22%5D%7D%2C%22position%22:%5B135%2C137%2C118%5D%2C%22crossSectionScale%22:1%2C%22projectionScale%22:512%2C%22layers%22:%5B%7B%22type%22:%22image%22%2C%22source%22:%22https://static.webknossos.org/misc/6001240.ozx%7Czip:%7Czarr3:%22%2C%22localDimensions%22:%7B%22c%27%22:%5B1%2C%22%22%5D%7D%2C%22localPosition%22:%5B0%5D%2C%22tab%22:%22source%22%2C%22opacity%22:1%2C%22blend%22:%22additive%22%2C%22shader%22:%22#uicontrol%20invlerp%20contrast%5Cn#uicontrol%20vec3%20color%20color%5Cnvoid%20main%28%29%20%7B%5Cn%20%20float%20contrast_value%20=%20contrast%28%29%3B%5Cn%20%20if%20%28VOLUME_RENDERING%29%20%7B%5Cn%20%20%20%20emitRGBA%28vec4%28color%20%2A%20contrast_value%2C%20contrast_value%29%29%3B%5Cn%20%20%7D%5Cn%20%20else%20%7B%5Cn%20%20%20%20emitRGB%28color%20%2A%20contrast_value%29%3B%5Cn%20%20%7D%5Cn%7D%5Cn%22%2C%22shaderControls%22:%7B%22contrast%22:%7B%22range%22:%5B7%2C927%5D%2C%22window%22:%5B0%2C1159%5D%7D%2C%22color%22:%22#ff0000%22%7D%2C%22volumeRenderingDepthSamples%22:256%2C%22name%22:%226001240.ozx%20c-0.5%22%7D%2C%7B%22type%22:%22image%22%2C%22source%22:%22https://static.webknossos.org/misc/6001240.ozx%7Czip:%7Czarr3:%22%2C%22localDimensions%22:%7B%22c%27%22:%5B1%2C%22%22%5D%7D%2C%22localPosition%22:%5B1%5D%2C%22tab%22:%22source%22%2C%22opacity%22:1%2C%22blend%22:%22additive%22%2C%22shader%22:%22#uicontrol%20invlerp%20contrast%5Cn#uicontrol%20vec3%20color%20color%5Cnvoid%20main%28%29%20%7B%5Cn%20%20float%20contrast_value%20=%20contrast%28%29%3B%5Cn%20%20if%20%28VOLUME_RENDERING%29%20%7B%5Cn%20%20%20%20emitRGBA%28vec4%28color%20%2A%20contrast_value%2C%20contrast_value%29%29%3B%5Cn%20%20%7D%5Cn%20%20else%20%7B%5Cn%20%20%20%20emitRGB%28color%20%2A%20contrast_value%29%3B%5Cn%20%20%7D%5Cn%7D%5Cn%22%2C%22shaderControls%22:%7B%22contrast%22:%7B%22range%22:%5B25%2C824%5D%2C%22window%22:%5B0%2C1025%5D%7D%2C%22color%22:%22#00ff00%22%7D%2C%22volumeRenderingDepthSamples%22:256%2C%22name%22:%226001240.ozx%20c0.5%22%7D%5D%2C%22selectedLayer%22:%7B%22visible%22:true%2C%22layer%22:%226001240.ozx%20c-0.5%22%7D%2C%22layout%22:%224panel-alt%22%2C%22helpPanel%22:%7B%22row%22:2%7D%2C%22settingsPanel%22:%7B%22row%22:3%7D%2C%22toolPalettes%22:%7B%22Shader%20controls%22:%7B%22side%22:%22left%22%2C%22row%22:1%2C%22query%22:%22type:shaderControl%22%7D%7D%7D) of this data has kindly been [made available](https://github.com/ome/ngff/pull/316#issuecomment-3302595684) by Davis Bennett.

Example scripts for writing OME-Zarr zip files in Python and Java, a checker for the requirements of this RFC and notes on the capabilities of the libraries used are provided in the `scripts/rfc9/` directory that accompanies this RFC.
Browser-based viewers need the server that hosts an OME-Zarr zip file to allow cross-origin range requests (the `Access-Control-Allow-Origin` header, and `Access-Control-Allow-Headers: Range` and `Access-Control-Expose-Headers: Content-Range, Content-Length, Accept-Ranges` for requests with a `Range` header).
The example datasets listed in the _Implementation_ section are served from a bucket that allows cross-origin requests.
