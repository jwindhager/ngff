# RFC-9: Response 1

(rfcs:rfc9:response1)=

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

We nevertheless want to actively encourage ZIP64.
We expect OME-Zarr datasets stored in zip files to keep growing.
ZIP64 is required once an archive exceeds 4 GiB, contains more than 65,535 entries, or has offsets beyond 4 GiB; the entry limit in particular can be reached by an unsharded Zarr hierarchy well below 4 GiB.
Using ZIP64 from the start lets datasets cross these thresholds smoothly, at a cost of a few bytes per entry that is small compared to common dataset sizes.
It also means that readers' ZIP64 code paths are exercised routinely rather than first encountered on large files.

We agree that guidance for common languages is needed, and will include library-specific notes for writing ZIP64 in the implementation guidance of the RFC.
We have therefore kept this as a SHOULD instead of downgrading it to a MAY, and have expanded the corresponding text in the _Proposal_ section to state the thresholds and the reason it is not a requirement.

#### Image preview

> The 'preview' aspect makes it tempting to want to embed a thumbnail, which could be supported by some applications or operating system plugins. Should this be explicitly forbidden / discouraged / encouraged in a standard way?

We agree that thumbnails are particularly relevant for single-file use cases.
However, we believe that thumbnail support applies to OME-Zarr in general and should therefore be proposed separately.

## Review 2

Response to [review 2](https://ngff.openmicroscopy.org/rfc/9/reviews/2/index.html) by Kola Babalola and Matthew Hartley, BioImage Archive, EMBL-EBI.

### Significant comments and questions

#### Versions of OME Zarr

> The RFC only applies to OME-Zarrs with metadata in zarr.json (not .zattr / .zarray) which implies at least OME Zarr v0.5. Is it worth explicitly mentioning this in the RFC? This might make sense in the ‘Compatibility’ section.

Agreed. We have added a clarifying note to the Compatibility section stating that this RFC applies only to OME-Zarr hierarchies with metadata in `zarr.json` (i.e. OME-Zarr v0.5 and later), and does not apply to hierarchies using the legacy `.zattrs`/`.zarray` metadata files (OME-Zarr v0.4 and earlier).

## Review 3

Response to [review 3](https://ngff.openmicroscopy.org/rfc/9/reviews/3/index.html) by Curtis Rueden, University of Wisconsin-Madison.

### Significant comments and questions

#### Advantages of ZIP

> It would be good for the RFC to break this down more explicitly, with a short discussion of each of ZIP's relevant advantages. That is: how good are each of these advantages in practice for OME Zarr?

Thank you for pushing us to spell this out; we expanded the Proposal section accordingly.

We also take your point on "simplicity" specifically: the ZIP file format itself, as laid out in the full APPNOTE specification, is not simple — it has accumulated many optional features (encryption, multiple compression methods, multi-volume archives, extra fields, etc.) over three decades.
What we meant, and should have stated more precisely, is that a lot of that complexity is already handled by mature, freely available libraries, so implementers of OME-Zarr zip file support rarely need to engage with the ZIP specification directly, let alone with the features this RFC does not recommend or prohibits.

In summary, our reasoning for choosing ZIP is primarily:

- **Widespread, low-effort implementation support.** ZIP has near-ubiquitous library support across programming languages, so adding `.ozx` support is comparatively cheap for existing OME-Zarr implementations, even though the underlying format is not simple.
  Several implementations (e.g. [zarr-python](https://zarr.readthedocs.io/en/stable/user-guide/storage/#zip-store), [tensorstore](https://google.github.io/tensorstore/kvstore/zip/index.html), [zarrita.js](https://zarrita.dev/packages/storage.html#zipfilestore)) already had ZIP support before this RFC, which we take as evidence that this low effort holds in practice, precisely because implementers could build on existing libraries rather than the specification itself.
- **Low effort scales from minimal to full-featured.** RFC-9 keeps the set of strict (MUST) requirements small, while the performance-related guidance (ZIP64, disabled compression, sharding, `jsonFirst` ordering) is RECOMMENDED rather than required.
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
The recommendation to use ZIP64 is meant to support this by allowing archives to continue growing.
We agree with the reviewer's observation that mutation has costs (a rewritten central directory, orphaned entries, and a possibly spoiled "`zarr.json` first" ordering, see the _Drawbacks_ section), and we have not adopted mitigations such as a padded header, in order to keep the format compatible with general-purpose ZIP libraries.
For massively parallel changes, it is expected that the Zarr arrays are extracted, modified, and then repacked.
We have stated this in the RFC.

##### Encryption

> It should be stated whether ozx files are allowed to use this feature or not, and if so, how use of encryption impacts other requirements.

Encryption at the archive level is not permitted.
We have added a MUST NOT requirement to the Specification section.
Encryption at the codec level may be applicable to Zarr in general, but we consider it out of scope for this RFC.

##### Recovery

> Is this a concern for ozx?

We consider it a limitation shared by ZIP in general.
Because ZIP also stores a header before each entry, tools can recover entries from a damaged archive by scanning forward.
We do not currently specify additional recovery mechanisms.
[Open: confirm wording with co-authors, see also the response to the recovery and performance recommendation.]

##### Performance and use cases

These are addressed in the responses to the performance and use case comments (see the corresponding sections of this document).

#### Explicitly constrain ZIP options

> Expand the "OME-Zarr zip files" section of the specification to more fully enumerate which ZIP features are permitted vs. forbidden (e.g., encryption not permitted; no extra fields beyond specified metadata; mutation permitted or not)

Thank you for the concrete proposal.
We have gone through the suggested list item by item:

- **Encryption (any method):** adopted for the ZIP archive level. OME-Zarr zip files MUST NOT use ZIP encryption. Codec-level encryption is out of scope for this RFC.
- **Multi-volume or split archives:** already a MUST NOT in the draft (requirement 4).
- **Compression at ZIP level:** adopted. OME-Zarr zip files MUST NOT use ZIP-level compression (STORE method only), since compression is expected to be performed by Zarr-level codecs. This may be relaxed in the future, e.g. for `zarr.json` documents.
- **Archive comments beyond the specified `ome` JSON:** not adopted as a prohibition on parameters. The archive comment is encoded as JSON precisely so that future versions of the specification can add parameters to it, for versioning and for parameterized features. Its content is defined by the specification under the `ome` attribute, and we have clarified this in the RFC. Comments that do not follow the specification remain permitted, since the comment is a recommendation (SHOULD). The `jsonFirst` parameter lets viewers discover the hierarchy without parsing the whole central directory, similar to the tree views of HDF5 viewers such as HDFView or h5web; we have expanded its description in the RFC.
- **Extra fields containing non-Zarr data** and **self-extracting ZIP code:** [Open: not yet decided.]
- **Mutation:** permitted, with the caveats described in the _Access and mutability_ subsection of the RFC (see also the response above).

Regarding "Validators MUST reject ozx files violating these constraints": we agree that the MUST requirements need to be testable. [Open: add validator language; ozx-tck is intended for this purpose.]

## Comment 1

Response to [comment 1](https://ngff.openmicroscopy.org/rfc/9/comments/1/index.html) by Matt McCormick, Fideus Labs LLC.

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

## Comment 3

Response to [comment 3](https://ngff.openmicroscopy.org/rfc/9/comments/3/index.html) by Chris Barnes, German BioImaging.

## Comment 4

Response to [comment 4](https://ngff.openmicroscopy.org/rfc/9/comments/4/index.html) by Lenard Spiecker and Matthias Grunwald, Miltenyi Biotec B.V. & Co. KG.

## Comment 5

Response to [comment 5](https://ngff.openmicroscopy.org/rfc/9/comments/5/index.html) by Anna Kreshuk, Dominik Kutra, and Dominik Kutra, Ilastik.

### Minor comments and questions

> The proposed new section of the specification uses the term "SHALL", which is so far not used elsewhere in the specification. Since according to IETF RFC 2119, SHALL is synonymous to MUST, and MUST is the term used in the rest of the specification, this should be replaced.

This suggestion has now been adopted.

> Duplication of "the" in "The ZIP file MUST contain the the OME-Zarr's root-level zarr.json."

This typo has now been corrected.
