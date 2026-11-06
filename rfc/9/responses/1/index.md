---
authors:
  - name: Jonas Windhager
    github: jwindhager
    affiliation: SciLifeLab / Uppsala University, Sweden
  - name: Norman Rzepka
    github: normanrz
    affiliation: scalable minds GmbH, Germany
  - name: Mark Kittisopikul
    github: mkitti
    affiliation: HHMI Janelia, United States
date: "2026-11-01"
---

# RFC-9: Response 1 (2025-12-11 version)

(rfcs:rfc9:response1)=

```{document-authors}
```

## Review 1

Response to [review 1](https://ngff.openmicroscopy.org/rfc/9/reviews/1/index.html) by Pete Bankhead, University of Edinburgh.

### Minor comments and questions

#### Use of ZIP64

> 1. The ZIP64 format extension SHOULD be used, irrespective of the ZIP file size.
>
> I'm not familiar enough with ZIP to understand the rationale for this recommendation or how straightforward it would be to follow.
>
> Specifically for Java, Zip files can be written with [`ZipFile`](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/zip/ZipFile.html) or the [optional Zip file system module](https://docs.oracle.com/en/java/javase/25/docs/api/jdk.zipfs/module-summary.html).
> I believe both support ZIP64, but I do not see an API to request that it is always used, including for smaller files. Apache Commons Compress [provides more ZIP64 control](https://commons.apache.org/proper/commons-compress/apidocs/org/apache/commons/compress/archivers/zip/ZipArchiveOutputStream.html#setUseZip64(org.apache.commons.compress.archivers.zip.Zip64Mode)), at the expense of requiring an extra dependency.
>
> The source for OpenJDK's `ZipFileSystem` [mentions a `"forceZIP64End"` property](https://github.com/openjdk/jdk/blob/master/src/jdk.zipfs/share/classes/jdk/nio/zipfs/ZipFileSystem.java#L179), but this appears to be undocumented.
>
> Will guidance / tooling be provided to achieve this recommendation in common languages?
> Otherwise, if it's technically hard to achieve and likely to be ignored in practice, might this be downgraded from SHOULD to MAY?

Thank you for raising this.
The recommendation is a SHOULD, not a MUST, so OME-Zarr zip files that do not use ZIP64 remain valid.
We acknowledge that not all ZIP libraries currently offer an option to write ZIP64 for small archives, which is the main reason ZIP64 is not required.

We nevertheless want to actively encourage ZIP64, and we have made the recommendation more precise.
The ZIP format itself requires ZIP64 once an archive reaches 65,535 entries (which an unsharded Zarr hierarchy can do well below 4 GiB) or once an archive, a central directory, an entry or an entry offset reaches 4 GiB.
The RFC now recommends that:

1. the ZIP64 end of central directory records SHOULD be present, irrespective of the size of the archive; and
2. entries that are or may become larger than 4 GiB SHOULD use the ZIP64 extra fields, while writers MAY use them for all entries.

The first recommendation costs about 76 bytes once per archive and makes it easier to append to an archive as it passes through the thresholds on the number of entries and on the size and offset of the central directory.
The second matters for entries such as shards, which can be large and whose final size may not be known when they are written.
Using the extra fields for all entries adds tens of bytes per entry: in the libraries we tested, 20 bytes per local file header and up to about 32 further bytes per central directory record.
We consider this worthwhile relative to the gigabyte to terabyte scale at which OME-Zarr datasets are expected to grow; for example, a 1 TB dataset stored as about 1,000 shards of 1 GB carries well under 100 kB of such overhead.

We agree that guidance for common languages is needed, and have written library-specific notes for writing ZIP64 as part of the implementation guidance that accompanies the RFC ([`library-notes.md`](https://github.com/jwindhager/ngff-rfc9-examples/blob/main/library-notes.md) in the [ngff-rfc9-examples](https://github.com/jwindhager/ngff-rfc9-examples) repository).
Libraries differ in which of these pieces they can write: for example, of the libraries we tested, Python's `zipfile` can write the extra fields in the local headers but not the end records for small archives, while the JDK's zip file system can write the end records (through an undocumented property) but not the extra fields.
We have therefore kept this as a SHOULD instead of downgrading it to a MAY, and have expanded the corresponding text in the _Proposal_ section to state what is required by the ZIP format itself, what we recommend, and why it is not a requirement.

#### Image preview

> The 'preview' aspect makes it tempting to want to embed a thumbnail, which could be supported by some applications or operating system plugins. Should this be explicitly forbidden / discouraged / encouraged in a standard way?

We agree that thumbnails are particularly relevant for single-file use cases.
However, we believe that thumbnail support applies to OME-Zarr, and to Zarr in general, and should therefore be proposed separately. A [Zarr convention for thumbnails](https://github.com/clbarnes/zarr-convention-thumbnails) has been proposed. The implementation of thumbnails and the integration of this convention into OME-Zarr are out of scope for this RFC.
We have made explicit in the RFC how such a convention can compose with OME-Zarr zip files: through metadata inside the Zarr hierarchy, and through a top-level key of its own in the archive comment next to `ome` (see the section on the archive comment structure).

## Review 2

Response to [review 2](https://ngff.openmicroscopy.org/rfc/9/reviews/2/index.html) by Kola Babalola and Matthew Hartley, BioImage Archive, EMBL-EBI.

### Significant comments and questions

#### Versions of OME Zarr

> The RFC only applies to OME-Zarrs with metadata in zarr.json (not .zattr / .zarray) which implies at least OME Zarr v0.5. Is it worth explicitly mentioning this in the RFC? This might make sense in the ‘Compatibility’ section.

Agreed. We have added the following sentence to the Compatibility section: "This RFC applies only to OME-Zarr v0.5 and later, which depend on Zarr version 3." Since Zarr version 3 stores metadata in `zarr.json`, hierarchies with `.zattrs`/`.zarray` metadata (OME-Zarr v0.4 and earlier) are not covered.

#### Recommendations on maximum archive size

> However, no explicit guidance is given on size limitations associated with the single file format. Presumably the upper limit of size of a single file on filesystems is a hard limit, but there are practical limitations to the storage and transfer of extremely large files below the filesystem-imposed limit. Since the RFC explicitly prohibits multi-part archives, it would be useful to include a brief discussion of the limitations this imposes, and guidance for users with OME-Zarrs above this size.

Thank you; we agree that this deserves discussion.
We do not give a fixed maximum size, because the practical limit depends on the file system, the object store and the way the file is transferred, and changes over time.
The hard bound is given by the 64-bit size limits of ZIP64, which are far beyond practical sizes.
In practice, very large single files are limited by file system and object store limits (for example the maximum size of a single object) and by the cost of transferring or copying one very large file, e.g. when a transfer has to be restarted.
We have added this as a drawback ("The size of a single file is bounded"), together with the guidance that a directory-backed or object store-backed OME-Zarr remains an option for very large datasets.

The review also points out a tension between the motivation of the RFC, which emphasizes sharing "a few small" images (_Background_), and the use of OME-Zarr zip files for transport and archival of large datasets (_Intended use cases and limitations_).
The motivation describes the use case that is served worst by directory-backed OME-Zarr today, but the format does not restrict the size of the hierarchy: terabyte-scale Zarr hierarchies are already distributed as single archives for transport and archival.
Whether a single file is appropriate for a given dataset is a practical decision for which the drawback above gives guidance.

### Minor comments and questions

> The specification section has the line "Amend the specification with the following section:" Is this a reference to the section that immediately follows?

Yes; we have reworded the sentence to make this explicit.

> The example json for the ome attribute in the zip archive has an extraneous comma on the line `"jsonFirst": true`

Thank you, the comma has been removed.

> The phrase: "These disadvantages were considered to be outweighed by other aspects (see Proposal section)." looks like it should be unindented so it applies to the whole "Drawbacks" section

Agreed; the sentence is now a separate paragraph after the list of drawbacks.

> Is the limitation (imposed by enforcing single-file archival) of the total size of an OME-Zarr image that can be represented as a single file this way a Drawback?

Yes, in practice: the limits of ZIP64 are far beyond practical sizes, but very large single files are difficult to handle (file system and object store limits, transfer and copy costs).
We have added this to the drawbacks, noting that directory-backed or object store-backed OME-Zarr remains an option for very large datasets (see the response on recommendations on maximum archive size above).

> Compatibility: Is it fully compatible with zarr < 0.5?

No: see the response above ("Versions of OME Zarr"); the RFC applies only to OME-Zarr v0.5 and later.

> Is the example dataset at https://static.webknossos.org/misc/6001240.ozx supposed to be accessible by viewers? [...] Access to fetch [...] has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.

> [Neuroglancer link] with acl to public and CORS to allow all origins works.

Thank you for testing this.
We have moved the file hosting to [IDR](https://livingobjects.ebi.ac.uk/idr/zarr/v0.5/idr0062A/ozx/6001240.ozx) and added a note on the required CORS configuration to the _Tutorials and Examples_ section, and added another set of 51 example datasets, hosted in a bucket that allows cross-origin requests, to the _Implementation_ section.

> Tutorials and Examples -> [...] A short worked example of how to write/construct .ozx would be valuable.

We agree; see the response on implementation guidance (Review 3) and the scripts that accompany the RFC.
We have referenced them in the _Tutorials and Examples_ section.

## Review 3

Response to [review 3](https://ngff.openmicroscopy.org/rfc/9/reviews/3/index.html) by Curtis Rueden, University of Wisconsin-Madison.

### Significant comments and questions

#### Advantages of ZIP

> It would be good for the RFC to break this down more explicitly, with a short discussion of each of ZIP's relevant advantages. That is: how good are each of these advantages in practice for OME Zarr?

Thank you for pushing us to spell this out; we expanded the Proposal section accordingly.

We also take your point on "simplicity" specifically: the ZIP file format itself, as laid out in the full APPNOTE specification, is not simple — it has accumulated many optional features (encryption, multiple compression methods, multi-volume archives, extra fields, etc.) over three decades.
What we meant, and should have stated more precisely, is that a lot of that complexity is already handled by mature, freely available libraries, so implementers of OME-Zarr zip file support rarely need to engage with the ZIP specification directly, let alone with the features this RFC does not recommend or prohibits.
We have rephrased the corresponding paragraph in the RFC accordingly, and now explicitly reference the APPNOTE specification, noting that OME-Zarr zip files only rely on features available since its version 4.5.

In summary, our reasoning for choosing ZIP is primarily:

- **Widespread, low-effort implementation support.** ZIP has near-ubiquitous library support across programming languages, so adding `.ozx` support is comparatively cheap for existing OME-Zarr implementations, even though the underlying format is not simple.
  Several implementations (e.g. [zarr-python](https://zarr.readthedocs.io/en/stable/user-guide/storage/#zip-store), [tensorstore](https://google.github.io/tensorstore/kvstore/zip/index.html), [zarrita.js](https://zarrita.dev/packages/storage.html#zipfilestore)) already had ZIP support before this RFC, which we take as evidence that this low effort holds in practice, precisely because implementers could build on existing libraries rather than the specification itself.
- **Low effort scales from minimal to full-featured.** RFC-9's strict (MUST) requirements mostly fix the layout and exclude ZIP features that every reader would otherwise have to support (e.g. encryption, ZIP-level compression, split archives), while the performance-related guidance (ZIP64, sharding, `jsonFirst` ordering) is RECOMMENDED rather than required.
  A minimal, spec-compliant reader/writer is therefore cheap to build, and [prototypes](https://github.com/clbarnes/ozx) have shown that a fully recommendation-compliant writer is feasible with only moderate additional complexity — i.e. there isn't a large cliff between "basic" and "full-featured" support.
- **Chunked access on local and remote stores alike.** The central directory enables efficient partial reads not just on local filesystems, but also on HTTP(S), S3, and GCS via range requests, which matches how OME-Zarr is already accessed today.
- **`.ozx` files are expected to be produced by OME-Zarr-aware tooling, not hand-zipped by end users.** We do not consider it a primary goal that users routinely feed `.ozx` files to a generic unzip tool, or that they modify them in place with generic ZIP tools; both remain possible but are not the design target.
  This is also why we did not design specifically around CRC32 integrity checking or post-creation mutability — the goal is to make it easy for tools that already understand OME-Zarr to add `.ozx` support, not to optimize for manual, ZIP-tool-based workflows.

#### Devil's advocate pitch for a minimal binary format over ZIP

> Conversely, a bespoke binary format minimizes unnecessary complexity, could embed structural metadata up front in a form tailored for rapid ingestion into an appropriate data structure of block offsets, and there would be only one needed code path for readers and writers.

Thank you for elaborating on this position.
We agree that a bespoke binary format would allow for more optimizations and would remove some of the "footguns" that ZIP's flexibility introduces; our MUST requirements are aimed at closing off exactly those unfortunate configurations without requiring a new format altogether.
Combined with `.ozx` files being expected to originate from OME-Zarr-aware tooling rather than generic zipping, we consider the risk of highly divergent, non-compliant files in the wild to be limited in practice.

Ultimately, though, we weigh this trade-off in favor of ZIP because widespread adoption is the primary goal of this RFC.
With a bespoke format, every language and toolkit that wants `.ozx` support would need to write, test, and maintain its own reader/writer from scratch.
ZIP, by contrast, already has mature libraries in most languages that absorb the bulk of that complexity, so most implementations get most of the way to compliant support for comparatively little effort.
We believe widespread adoption depends on this low barrier to entry.
That said, we agree the complexity/adoption trade-off is ultimately an empirical question, and we will need to monitor the ecosystem as adoption grows.

To be clear about what this RFC defines: it is a file format.
An OME-Zarr zip file (`.ozx`) is a valid ZIP file whose unzipped contents are a valid OME-Zarr hierarchy, restricted by a small set of MUST requirements and a set of recommendations that allow for performance and future parameterization via the archive comment.
It is a conservative subset of all possible "Zarr in a ZIP file" layouts, and therefore its own format; we have added a section to the RFC that says so.

#### Requirements: streaming, mutability, encryption, recovery

> The proposal should articulate the technical requirements of a single-file OME-Zarr format more thoroughly.

Thank you; we agree.
We have added an _Access and mutability_ subsection to the Proposal section and a requirement on encryption to the Specification section.

##### Streaming

> Is ozx intended to be streamable from a remote source?
> - If not: this should be stated explicitly in the RFC that streamability is a non-goal.

Streaming is not a goal.
Like the rest of Zarr, network access is intended to use range-read requests, which are available from standard HTTP servers and object storage services such as S3.
Client software is expected to retrieve the central directory without reading the entire archive.
We have stated this explicitly in the RFC.
Regarding the question on `jsonFirst`: it does not make offsets of all chunks available up front and is not intended to; it lets readers discover the hierarchy from the start of the central directory without parsing all of it.

##### Mutability

> Are the contents of a zipped OME-Zarr file intended to be mutable?

Some mutability is expected: the archive can be appended to, files can be modified in-place, and the central directory can be rewritten to omit obsolete data or files.
Comment 5 recommends treating OME-Zarr zip files as read-only after the initial write and rewriting the whole file for any modification. We have not adopted this recommendation, because the recommendations of the RFC (ZIP64, writing the central directory at the end of a writing session, removing duplicate records) set up the archive to allow for some updates of the data (see the response to comment 5 below).
We agree with the reviewer's observation that mutation has costs (a rewritten central directory, orphaned entries, and a possibly spoiled "`zarr.json` first" ordering, see the _Drawbacks_ section), and we have not made mitigations such as padding a requirement, in order to keep the format compatible with general-purpose ZIP libraries; padding by means of alignment and padding extra fields, or by reserving unused space between entries, is permitted.
For massively parallel changes, it is expected that the Zarr arrays are extracted, modified, and then repacked.
We have stated this in the RFC.

##### Encryption

> It should be stated whether ozx files are allowed to use this feature or not, and if so, how use of encryption impacts other requirements.

Encryption of ZIP entries or of the central directory is not permitted.
We have added a MUST NOT requirement to the Specification section.
Encryption at the codec level may be applicable to Zarr in general, but we consider it out of scope for this RFC.

##### Recovery

> Is this a concern for ozx?

Like other archive formats with a trailing index, a ZIP archive that is corrupted or only partially transferred is hard to use directly, because the central directory is at the end of the file.
Since every entry is also preceded by a local file header, and since this RFC requires that no data precedes the first local file header, recovery tools can often scan forward through the archive and rebuild the central directory.
This is not guaranteed, e.g. when entries were written with data descriptors (sizes recorded after the entry data), because the end of such an entry cannot be found without the central directory.
Recovery is not a goal of this RFC, and we do not specify additional recovery mechanisms. We have stated this in the RFC.

##### Performance

> Is fast performance a goal of this format? If so, how fast? [...] Can that be achieved well with a single-file zipped OME-Zarr structure? How does it compare to unzipped OME-Zarr?

Yes: we expect reading OME-Zarr zip files to be comparable to reading other Zarr stores when readers use range requests and the recommendations are followed.
We have rewritten the _Performance_ section of the RFC to summarize the most relevant findings of the existing external evaluations (see also the response to comment 5 below), and to state this expectation.
We have measured central directory costs on synthetic archives (scripts in [ngff-rfc9-examples](https://github.com/jwindhager/ngff-rfc9-examples)): parsing time and central directory size grow linearly with the number of entries (about 0.6 s and 59 MB at one million entries with a minimal parser), lookups after indexing are independent of the number of entries, and with `jsonFirst` the hierarchy is discovered in time independent of the number of entries. These measurements are local and do not include network latency, and the latency of random-access chunk reads compared to an unzipped store has not been measured. We have summarized these results in the _Performance_ section of the RFC.
The _Drawbacks_ section already describes the logic that readers may need to optimize lookups (parsing the central directory into a hash table or sorting it).

##### Use cases

This is addressed in the response to comment 3 (see the corresponding section of this document).

#### Generalizing `jsonFirst`

> For future-proofing, I suggest generalizing this field beyond only a boolean. It would make sense as a field defining the nature of the tree structure. Something like "treeStructure": "levelOrder" (i.e. breadth first).

`jsonFirst` asserts a property of the set of records, not of their order: every `zarr.json` record precedes every other record in the central directory, not counting directory entries.
This is exactly the property a reader needs in order to stop parsing the central directory early and still know that the structure of the hierarchy is complete; it does not depend on any particular order among the `zarr.json` records.
A separate flag for breadth-first ordering was proposed during the drafting of the RFC and later removed for this reason.
We have clarified this in the RFC (including that `jsonFirst` does not assert breadth-first order), and we have added an example.

> It would be helpful for the proposal to give an example of central directory size vs zarr.json size, to give a sense of performance gains here.
> - For the example ozx file?
> - For a larger file, e.g. tubhiswt.ome.tif converted to ozx with a reasonable sharding structure?

For the example dataset [`6001240.ozx`](https://livingobjects.ebi.ac.uk/idr/zarr/v0.5/idr0062A/ozx/6001240.ozx), the central directory has 108 records in 6,154 bytes; the 4 `zarr.json` records come first and take 226 bytes. This dataset is too small for `jsonFirst` to make a noticeable difference, and its archive comment (`{"ome": {"version": "0.5"}}`) does not set it.
The gain grows with the number of entries: on synthetic archives, the size of the central directory grows linearly with the number of entries (about 59 MB at one million entries, parsed in about 0.6 s with a minimal parser), whereas with `jsonFirst` a reader only needs the leading `zarr.json` records, whose size depends on the number of groups and arrays but not on the number of chunks or shards (see the response on performance above and the _Performance_ section of the RFC).
We have not converted a larger, realistically sharded dataset for this response.
Since the keys under `ome` are strictly defined by the specification and are extended by future versions of it, an additional parameter describing the order of the `zarr.json` records can be added later should a use case arise; readers that do not understand it are asked to ignore it.

#### Explicitly constrain ZIP options

> Expand the "OME-Zarr zip files" section of the specification to more fully enumerate which ZIP features are permitted vs. forbidden (e.g., encryption not permitted; no extra fields beyond specified metadata; mutation permitted or not)

Thank you for the concrete proposal.
We have gone through the suggested list item by item:

- **Encryption (any method):** adopted. OME-Zarr zip files MUST NOT encrypt ZIP entries or the central directory. Codec-level encryption is out of scope for this RFC.
- **Multi-volume or split archives:** already a MUST NOT in the draft (requirement 4).
- **Compression at ZIP level:** adopted. OME-Zarr zip files MUST NOT use ZIP-level compression (STORE method only), since compression is expected to be performed by Zarr-level codecs. This may be relaxed in the future, e.g. for `zarr.json` documents. We considered allowing readers to process generic ZIP files that do not comply with this RFC, e.g. archives created with the default tooling of an operating system (which typically use DEFLATE and place the hierarchy in a folder) and renamed to `.ozx`, and decided against it, since the `.ozx` extension asserts compliance; readers remain free to support plain `.zip` files. We have recorded this in the _Abandoned Ideas_ section of the RFC.
- **Archive comments beyond the specified `ome` JSON:** partly adopted. The archive comment is encoded as JSON precisely so that future versions of the specification can add parameters to it, for versioning and for parameterized features. The keys under the top-level `ome` attribute are strictly defined by the specification, and we have clarified this in the RFC; readers SHOULD ignore keys under `ome` they do not understand, so that keys added in later versions do not break older readers. Other top-level keys are permitted, to allow composition with other specifications. The `jsonFirst` parameter lets viewers discover the hierarchy without parsing the whole central directory, similar to the tree views of HDF5 viewers such as HDFView or h5web; we have expanded its description in the RFC.
- **Extra fields containing non-Zarr data:** partly adopted.
    - Most extra fields carry per-entry metadata rather than Zarr data (e.g. ZIP64 information, timestamps, Unix permissions), and a validator cannot tell what an unknown extra field contains. We therefore do not prohibit extra fields in general; readers can skip those they do not understand.
    - Instead, requirement 7 prohibits extra fields that change how the bytes of an entry are to be interpreted.
    - Extra fields known to be compliant (header IDs from APPNOTE 6.3.10; these lists are also given in the _Access and mutability_ subsection of the RFC):
        - `0x0001`: ZIP64 extended information.
        - `0x000a`, `0x5455`: NTFS and extended timestamps.
        - `0x5855`, `0x7855`, `0x7875`: Info-ZIP UNIX owner and permission information.
        - `0xa11e`, `0xa220`, `0xcafe`, `0xd935`: alignment and padding (Commons Compress, Microsoft Open Packaging, Java JAR, Android).
        - `0x7075`, `0x6375`: Info-ZIP Unicode Path and Unicode Comment. They store the UTF-8 form of the file name and comment in the header, so they do not change which Zarr node an entry belongs to.
    - Extra fields known to be non-compliant:
        - `0x9901`: AE-x (WinZip AES) encryption structure.
        - `0x0017`: Strong Encryption Header.
        - `0x000f`: Patch Descriptor (the entry holds a patch against another file, not the content of a Zarr key).
    - Records that carry encryption keys and certificates (`0x0019`, `0x0021`, `0x0022`, `0x0023`) accompany encryption, which is already prohibited.
- **Self-extracting ZIP code:** adopted in a general form. OME-Zarr zip files MUST NOT contain any data before the first local file header, which prohibits self-extracting archives and other prepended stubs. This may be revisited in a future version of the specification.
- **Mutation:** permitted, with the caveats described in the _Access and mutability_ subsection of the RFC (see also the response above).

Regarding "Validators MUST reject ozx files violating these constraints": we agree that the MUST requirements need to be testable and have written them with that in mind. We have not added a requirement on validators themselves, since conformance of files is already defined by the MUST and MUST NOT requirements, and requirements on validators are an implementation detail that the RFC does not otherwise define. Requirement 7 is testable in part: the lists of compliant and non-compliant extra fields above can serve as the basis for a test. [ozx-tck](https://github.com/clbarnes/ozx-tck) is intended for this purpose.

#### Provide implementation guidance

> **Provide implementation guidance**: Show concrete examples of constructing ozx files, demonstrate performance characteristics, and highlight some languages/libraries that meet the spec's constraints.

We have added implementation guidance as scripts and notes that accompany the RFC (the [ngff-rfc9-examples](https://github.com/jwindhager/ngff-rfc9-examples) repository):

- **Examples of constructing .ozx files**: self-contained writers for Python with the standard library `zipfile` and with the third-party package `stream-zip`, and for Java with the JDK alone and with Apache Commons Compress. They are tested against the checker described below.
- **A checker** (`check_ozx.py`) that tests the MUST requirements of this RFC on a local file or a URL using range requests. It found no violations of the checked requirements in the 51 example datasets listed in Zipglancer.
- **Library notes** ([`library-notes.md`](https://github.com/jwindhager/ngff-rfc9-examples/blob/main/library-notes.md)) that state, for each library tested, which of the requirements and recommendations it can meet and where it falls short, e.g. the lack of control over the central directory order, the lack of an archive comment in the JDK zip file system and in `stream-zip`, and differences in how libraries write ZIP64 (see also the ZIP64 recommendations in the RFC). The RFC lists the libraries and tools that support OME-Zarr zip files in its _Implementation_ section.
- **Performance characteristics**: the _Performance_ section summarizes external evaluations, and a benchmark script measures central directory costs (see the response on performance).

### Minor comments and questions

#### Abandoned Ideas section

> The "Semantically restrict the contents of single-file OME-Zarr" point makes sense, but I would add that this idea is orthogonal to that of a zipped single-file format: one could just as well propose semantic restrictions on multi-file OME-Zarr, and it could potentially have value, but such a discussion should be outside the scope of this RFC.

Agreed; we have added this to the corresponding item in the _Abandoned Ideas_ section.

## Comment 1

Response to [comment 1](https://ngff.openmicroscopy.org/rfc/9/comments/1/index.html) by Matt McCormick, Fideus Labs LLC.

### ZIP comment flag for file ordering

> We agree with [the suggestion](https://github.com/ome/ngff/pull/364) to include a flag in the ZIP comment to indicate whether this ZIP ordered the files as suggested for clients. This would help readers optimize their parsing strategy.

Thank you; this flag is `jsonFirst` in the archive comment, which is already part of the proposal.
We have clarified in the RFC that it asserts that all `zarr.json` records precede all other records in the central directory (directory entries, which may appear anywhere, are not counted), and that writers that modify an archive SHOULD keep the flag consistent with the order of the central directory.
[Zipglancer](https://github.com/JaneliaSciComp/zipglancer), a web-based explorer for ZIP and .ozx archives, reads this setting, and we have added it to the list of implementations.

### File order example

> We recommend including a concrete example of the expected file order for clarification.

Agreed; we have added an example to the RFC.
Note that the example shows the central directory order with the recommended breadth-first order of the `zarr.json` records, which `jsonFirst` itself does not require.

### Collections RFC reference

> The sentence:
> 
> > This restriction may be revised in the future, especially in the light of the "collections" RFC.
> 
> is confusing in this context. The "collections" RFC is not yet defined, and the connection to allowing embedded OME-Zarr zip files is unclear. We suggest removing this sentence to avoid confusion.

This sentence has now been removed.

### Minor comments and questions

> There was a comment about making the ZIP comment null-terminated. This is extraneous and should be removed.

This was indeed erroneous and had initially been [removed](https://github.com/ome/ngff/pull/316/changes/75d0f4640c1685e07d7fce2d33f36ee1ebd6b5a3).
An accidental [regression](https://github.com/ome/ngff/pull/316/commits/657b3345b82997b2cd5cd0dc17eca7ed93e2d342) occurred during the merge, which has since been [hot-fixed](https://github.com/ome/ngff/commit/355530782eb2405a9b1feaea331ebe47cbbe6033) in the current version.

## Comment 2

Response to [comment 2](https://ngff.openmicroscopy.org/rfc/9/comments/2/index.html) by Joost de Folter, BioImaging-NL.

### Minor comments and questions

> Can one use ZIP64 on files that don't require it? It appears that ZIP64 records only exist in zip files that pass the 64bit length boundary. If the force_zip64 flag resolves this, then this would require this set as default in ZipStore I assume?

Yes, ZIP64 can be used for archives that do not require it, and the RFC now recommends this in two parts (see the response on ZIP64 above).
Of the libraries we tested, Python's `zipfile` can write the ZIP64 extra field in the local headers of every entry with `force_zip64=True`, but it cannot force the ZIP64 end records for a small archive.
The `ZipStore` of zarr-python 3.4.1 does not use ZIP64 for small archives (none of the 22 local headers in our test had the field), so following the recommendation would require support in `ZipStore`.

> https://github.com/python/cpython/issues/47073 - conceptually, data and metadata needs to be written together (add to implementation section)

Thank you; we have added a note to the _Implementation_ section.
Python's `zipfile` permits several entries with the same name, so rewriting a `zarr.json` produces duplicate central directory records, which the RFC now recommends to remove at the end of a writing session (see also the notes on `ZipFile.remove()` and `ZipFile.repack()`, planned for Python 3.16, in the implementation guidance).

## Comment 3

Response to [comment 3](https://ngff.openmicroscopy.org/rfc/9/comments/3/index.html) by Chris Barnes, German BioImaging.

### Minor comments and questions

#### Use case documentation

> it will be important for the eventual spec changes to make clear the use cases for which .ozx is and is not appropriate. e.g. as a write-once read-many format for small datasets for archival or transport purposes.

We agree and have added a section on _Intended use cases and limitations_ to the RFC.
It lists the intended use cases (file-centric desktop workflows, transport and archival, viewing and exploring remote data with range requests, and exchange between Zarr implementations), points to current demonstrations of them, and lists what OME-Zarr zip files are not intended for (many concurrent writers or frequent in-place modification, streaming, encryption of ZIP entries or the central directory, and very large numbers of entries without sharding).
The format is primarily write-once read-many, as suggested, but not exclusively: some mutability is expected (see the response to the comment on mutability above).
Unlike the comment's example, we do not restrict the intended use to small datasets: transport and archival of large datasets as a single object is one of the intended use cases, with the practical limits of very large single files described in the _Drawbacks_ section (see also the response on maximum archive size).

The demonstrations we found in the record of this RFC and elsewhere are:

- the Neuroglancer and Vizarr views of the example dataset `6001240.ozx` (see the _Implementation_ and _Tutorials and Examples_ sections);
- WEBKNOSSOS, which can read OME-Zarr zip files remotely and uploaded ones ([pull request](https://github.com/scalableminds/webknossos/pull/9738), merged);
- Zipglancer, a web-based explorer for ZIP and OME-Zarr zip files using range requests, which provides links to 51 example OME-Zarr zip files in a public bucket (all of which pass the checks for the requirements of this RFC that `check_ozx.py` can perform on the central directory, and which all use the `jsonFirst` ordering);
- converters and test tools: the Rust converter `ozx` and the validator `ozx-tck`.

This list is not exhaustive. The use case that initially motivated this RFC, opening and sharing OME-Zarr images in file-centric desktop workflows, is described in the _Background_ section.

#### Beyond OME-Zarr

> The recommendations made by the RFC (zarr.json at the root, sorted metadata files) are valuable to all single-file Zarr users, and the only OME-Zarr specific elements are arbitrary markers (the ZIP comment and file extension). [...] Would it be possible to submit this meta-format as e.g. a [zarr convention](https://zarr.dev/conventions/), with more generic markers?

Thank you for the suggestion.
We would like to continue to iterate on these recommendations in the context of OME-Zarr before addressing them to Zarr directly.
A version of this proposal could become a Zarr convention, and that could be a way to iterate on it more quickly than the OME-Zarr RFC process allows.
We have added this to the _Future possibilities_ section.
In the meantime, the design leaves room for this: the keys under `ome` in the archive comment are defined by this specification, while other top-level keys are permitted to allow composition with other specifications.

## Comment 4

Response to [comment 4](https://ngff.openmicroscopy.org/rfc/9/comments/4/index.html) by Lenard Spiecker and Matthias Grunwald, Miltenyi Biotec B.V. & Co. KG.

### Minor comments and questions

#### Ordering of zarr.json first

> While placing the root and all other `zarr.json` files at the beginning of the archive potentially aids discovery and streaming access, practical implementations may still read the ZIP comment together with the central directory first. [...] We also observed that strict file ordering cannot be maintained when appending a new `zarr.json` (e.g., adding labels) to an existing .ozx file. Furthermore, we encounter cases where metadata is generated during acquisition; therefore, we lean toward writing data first and metadata second to avoid writing it twice.

Thank you for this detailed feedback; it showed us that the RFC conflated two different orders.
The purpose of the ordering is not streaming (which is not a goal of this RFC, see above) but to let applications show the structure of the hierarchy, for example as a tree view, without parsing the entire central directory.
This only requires the order of the records in the central directory, not the order in which the entries are stored in the archive.
The central directory is rewritten whenever an archive is appended to, so it can be put into the recommended order at the end of a writing session, regardless of when each `zarr.json` was written.
Data can therefore be written first and metadata second.
We have changed the RFC accordingly: the recommendation now concerns only the central directory, and the order of the entries themselves is not restricted.
For writers that cannot reorder the central directory, `jsonFirst` can be set to `false` (or omitted), which is valid.
As the comment suggests, readers may also read the archive comment and the central directory first; this remains possible.
Regarding duplicate `zarr.json` entries observed when appending: the RFC recommends that writers remove duplicate records from the central directory at the end of a writing session, and requires readers to use the last record if duplicates remain, as Python's `zipfile` does. A reader that relies on `jsonFirst: true` to stop reading early can still miss a later duplicate `zarr.json`; writers are therefore asked to keep the `jsonFirst` flag consistent with the central directory.

> This is reasonable for discoverability, especially for the root `zarr.json` if consolidated metadata is present. For all other `zarr.json` files, and for a root `zarr.json` without consolidated metadata, this seems less relevant for us and depends on the number of file entries. Therefore in our use case we might omit it for now and introduce it later if needed.

Agreed. The order of the central directory is a recommendation, and files that do not follow it remain valid; in that case `jsonFirst` is `false` or omitted.
The order matters most for archives with many entries, where it lets readers discover all `zarr.json` records without reading the entire central directory.
Consolidated metadata is not part of the OME-Zarr specification and is therefore not mentioned in the RFC, but the two are complementary: with the recommended order, the root-level `zarr.json`, including any consolidated metadata it holds, is the first record in the central directory.

#### ZIP disadvantage in performance

> Compared to a directory store, file content is not necessarily stored page-aligned. [...] This point could be added under the drawback section in the RFC.

Thank you for sharing this experience.
We have added the lack of page alignment, the resulting read-modify-write cycles for unbuffered page-aligned I/O, and the mitigation of allocating a separate page per local file header (at the cost of additional space, acceptable with sharding) to the _Drawbacks_ section of the RFC.
Besides leaving unused space between entries, as described in the comment, padding can also be achieved by means of alignment and padding extra fields, which are permitted by this RFC (see the requirement on extra fields).

#### Sharding constraints

> [...] partial writes are impractical unless the final size of each shard is known in advance. As a result, it is often not recommended to shard an axis that is acquired sequentially. [...] This point could be added under the drawback section in the RFC. The recommendation to SHOULD use sharding depends of course on the chunk size, the expected number of chunks and the codec pipeline.

Agreed; we have added this to the _Drawbacks_ section and made the dependence of the sharding recommendation on these factors explicit.

#### CRC/hash requirement

> ZIP requires CRC32 for file entries, which is useful for integrity verification, but it burdens implementations, especially for partial writes and reads. [...] clarifying recommended strategies (e.g., validating at shard or chunk granularity and deferring CRC checks for in-flight writes) would help implementers. [...] This point could be added under the drawback section in the RFC.

We have added this to the _Drawbacks_ section, including the observation on hardware support for CRC-32C but not for the CRC-32 used by ZIP, which we attribute to the comment.
The _Drawbacks_ section now states, as implementation guidance and not as a requirement, that implementations may validate at the granularity of shards or chunks and defer CRC-32 checks for entries that are still being written. This is an allowance, not a recommendation, and it is not part of the specification text.

#### ZIP disadvantage when updating

> [...] we allowed in-place updates as long as the size does not grow beyond the existing space. As an example, we added capacity (padding) to allow in-place updates of metadata. (Similar to `tiffcomment` or `tiffset` on tiff files). [...] We just wanted to mention here how this could be mitigated in certain cases.

Thank you; we have added this mitigation to the _Drawbacks_ section.
Padding is possible either by means of alignment and padding ZIP extra fields, which are permitted by this RFC (see the requirement on extra fields), or by reserving unused space between entries.

#### Split archives

> [...] we see use cases where archive-level splitting would be beneficial, particularly from a user-experience perspective. However, we acknowledge that this adds complexity to implementations, and support this decision.

Thank you for your support of this decision.
Splitting remains a possible future extension; see the _Future possibilities_ section.

#### Thumbnails

> Applications might benefit from pre-rendered thumbnails. [...] it might be a question if this should be a topic to be addressed by zipped OME-Zarr separately or if this is out of scope for this RFC.

We consider this out of scope for this RFC. Thumbnails apply to Zarr in general and are addressed by a [Zarr convention for thumbnails](https://github.com/clbarnes/zarr-convention-thumbnails), which has been proposed. The implementation of thumbnails and the integration of the convention into OME-Zarr are out of scope for this RFC; the latter is listed in the _Future possibilities_ section.
We have made explicit in the RFC how such a convention can compose with OME-Zarr zip files: through metadata inside the Zarr hierarchy, and through a top-level key of its own in the archive comment next to `ome`, which readers that do not understand it are expected to ignore.
The Open Packaging Conventions mentioned in the comment store thumbnails as separate parts of the package; OME-Zarr zip files may likewise contain entries that are not Zarr keys, since Zarr itself does not prohibit them (see the RFC).
This specification governs only the keys under `ome` in the archive comment; other top-level keys are not governed by it and are meant for composition.

#### Recommend specific implementations

> In general, the specification could recommend in the end using specific implementations over standard ZIP writers so that end users can create compatible .ozx files and avoid interoperability issues.

We agree with the intent: the RFC states that OME-Zarr zip files are expected to be produced primarily by OME-Zarr-aware tooling rather than by generic ZIP tools, and the _Implementation_ section lists the tools and libraries that support them.
We did not make a recommendation for specific implementations part of the specification, since this would tie the specification to the current state of the ecosystem; instead, the implementation guidance that accompanies the RFC (scripts, a checker, and notes on the capabilities of several libraries) helps implementers and users to create and verify compatible files.

## Comment 5

Response to [comment 5](https://ngff.openmicroscopy.org/rfc/9/comments/5/index.html) by Anna Kreshuk, Dominik Kutra, and Benedikt Best, Ilastik.

### Significant comments and questions

#### Clarify performance expectations

> For an RFC to a public standard, in our opinion, at least the most relevant information from the external source should be reflected in the text.

Agreed.
The _Performance_ section now summarizes the most relevant findings of the two external sources in the text, including the key numbers and the observation that the reader implementation matters more than the use of ZIP itself, and notes that those evaluations concern geospatial rather than microscopy data.

> "When creating OME-Zarr zip files, the following RECOMMENDATIONS ensure that reading from OME-Zarr zip files is similarly performant as reading from other storage formats:"

> "Writer implementations SHOULD verify that reading their OME-Zarr zip files is similarly performant as reading from other storage formats."

We adopted a variant of the first phrasing for the list of recommendations, stating that they are intended to ensure reading performance similar to other storage formats.
We did not adopt the second, stronger formulation as a normative requirement on writers, since writers cannot fully control reader implementations and such a requirement could not be tested on a file; instead, the _Performance_ section advises writers to verify read performance for their data.

#### Specify possible roots

> We recommend explicitly specifying the possible roots, since RFC-9 assigns meaning to the "root of the OME-Zarr hierarchy". For example: "The ZIP file MUST contain exactly one multiscale image (including optionally one labels group), or exactly one high-content screening dataset." At a minimum, we recommend replacing the word "hierarchy" with the equally broad "dataset" or "fileset" to avoid increasing the number of undefined terms in the specification.

Thank you for raising this.
We compared the terms with the current OME-Zarr specification.
"Zarr hierarchy" is already used there for the tree of Zarr groups and arrays below a root group.
"Dataset" is not better defined: in the multiscales metadata, `datasets` denotes the arrays that store the individual resolution levels, and elsewhere the word is used informally (for example "high-content screening dataset" and "scene dataset"); "fileset" is used in the specification for the layout produced by `bioformats2raw`.
We have therefore kept "hierarchy", and the RFC now defines the term "OME-Zarr hierarchy" once, as a Zarr hierarchy whose contents conform to the OME-Zarr specification.

Regarding the possible roots: this RFC defers them to the OME-Zarr specification as a whole and neither adds restrictions nor adds possibilities. It does not restrict the contents of the OME-Zarr that is stored, semantically or otherwise (see the item on semantic restrictions in the _Abandoned Ideas_ section), so the root of the archive is the root of whatever OME-Zarr hierarchy the specification permits, now or in the future.
This avoids tying the single-file format to the set of root types of a particular version of the specification, which may be extended by the collections RFC.

#### Avoid appending

> We suggest adding an explicit recommendation, such as: "It is RECOMMENDED that OME-Zarr zip files are treated as read-only objects after the initial write operation. Modifying operations SHOULD be implemented by rewriting the entire file."

Thank you; the difficulty of parallel writes into an archive is exactly what this comment highlights.
We have not adopted a recommendation to treat OME-Zarr zip files as read-only, because some mutability is expected and the recommendations of the RFC (ZIP64, writing the central directory at the end of a writing session, removing duplicate records) set up the archive to allow for some updates of the data.
At the same time, we acknowledge that an unzipped OME-Zarr may provide better write performance, especially when parallelization is involved; for massively parallel changes the arrays are expected to be extracted, modified, and repacked.
We have added both points to the _Access and mutability_ subsection of the RFC.

### Minor comments and questions

> The proposed new section of the specification uses the term "SHALL", which is so far not used elsewhere in the specification. Since according to IETF RFC 2119, SHALL is synonymous to MUST, and MUST is the term used in the rest of the specification, this should be replaced.

This suggestion has now been adopted.

> Duplication of "the" in "The ZIP file MUST contain the the OME-Zarr's root-level zarr.json."

This typo has now been corrected.

## Comment 6

Response to [comment 6](https://ngff.openmicroscopy.org/rfc/9/comments/6/index.html) by Assa Diabira, Institut Cochin (IMAG'IC / CID), Université Paris Cité, France.

### Significant comments and questions

#### Disabled ZIP compression is critical for random-access readers

> From a reader's standpoint, the “disabled compression” recommendation is therefore close to a usability requirement for browser side random access rather than a minor tuning knob, and I would support stating it more strongly.

Thank you for this detailed account from a browser-side reader.
Since the version of the RFC that the comment refers to, the recommendation has become a requirement: ZIP-level compression MUST NOT be used, and all entries MUST use the STORE method (requirement 6 in the _Specification_ section).
The reasons given in the RFC, avoiding the compression of already compressed Zarr chunks and allowing direct partial reads of the archive, match the experience described in the comment.

#### Root alignment

> Making root alignment a hard requirement in the RFC would allow readers to drop this detection step entirely.

This is already a hard requirement: the root of the ZIP archive MUST correspond to the root of the OME-Zarr hierarchy, and the root-level `zarr.json` MUST be located in the root of the archive (requirement 2 in the _Specification_ section).
Readers of OME-Zarr zip files therefore do not need to detect the root.
Archives in which the hierarchy is nested in a folder, e.g. because a directory was zipped with generic tooling, are not OME-Zarr zip files; readers may still support them as plain `.zip` files as an implementation choice (see the _Abandoned Ideas_ section of the RFC).

#### `jsonFirst` reader fallback

> One small clarification I would welcome is an explicit statement of what a reader may assume when `jsonFirst` is absent or false, presumably a fall back to a full central-directory scan.

Yes. The RFC states that `jsonFirst` defaults to `false` if it is omitted, and that without `jsonFirst` set to `true`, a reader has to parse the entire central directory to be sure that the whole structure of the hierarchy has been discovered.
A reader may therefore make no assumption about the order of the records in this case.

### Minor comments and questions

#### Upstream v2 tooling raises a repackaging cost for `.ozx` adoption

Thank you for pointing this out.
As stated in the _Compatibility_ section, this RFC applies only to OME-Zarr v0.5 and later, which depend on Zarr version 3 (see also the response on versions of OME-Zarr to Review 2).
We agree that data written as OME-Zarr v0.4 has to be converted to v0.5 before it can be stored as an OME-Zarr zip file; this conversion is not specific to this RFC.

#### Single-file archives usefully carry non-image members

We agree that bundling tables and labels with the image is one of the benefits of a single file.
The RFC does not restrict the contents of the OME-Zarr hierarchy that is stored, semantically or otherwise: any group or array that the OME-Zarr specification permits, such as a `tables` group written as Zarr, is stored like any other part of the hierarchy.
In addition, OME-Zarr zip files may contain entries that are not Zarr keys, since Zarr itself does not prohibit them.

## Further feedback

### ZIP-level compression may be useful

> Compression may still be useful for the zarr.json files. ([comment on a related pull request](https://github.com/ome/ngff/pull/364#issuecomment-3543196447))

Thank you.
At this time, OME-Zarr zip files MUST NOT use ZIP-level compression (STORE only), to keep readers and writers simple and partial reads direct.
We agree that compression of `zarr.json` documents could be useful, and the RFC states that this restriction may be relaxed in a future version of the specification for these documents; we have added it to the _Future possibilities_ section.

### Configurable root may be useful

> (feedback referring to SpatialData and to the HCS specification)

We have decided to defer a configurable root, i.e. locating the OME-Zarr hierarchy at a path inside the archive, to a future version of the specification, and have added it to the _Future possibilities_ section.
For now, the root of the ZIP archive MUST correspond to the root of the OME-Zarr hierarchy; this avoids additional prompts for users and inconsistencies when renaming OME-Zarr zip files.
If the root is made configurable in the future, this could be done through a parameter in the archive comment (the keys under `ome` are defined by the specification and extended only by future versions of it).

