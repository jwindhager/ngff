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

We nevertheless want to actively encourage ZIP64, and we have made the recommendation more precise.
The ZIP format itself requires ZIP64 once an archive reaches 65,535 entries (which an unsharded Zarr hierarchy can do well below 4 GiB) or once an archive, a central directory, an entry or an entry offset reaches 4 GiB.
The RFC now recommends that:

1. the ZIP64 end of central directory records SHOULD be present, irrespective of the size of the archive; and
2. entries that are or may become larger than 4 GiB SHOULD use the ZIP64 extra fields, while writers MAY use them for all entries.

The first recommendation costs about 76 bytes once per archive and makes it easier to append to an archive as it passes through the thresholds on the number of entries and on the size and offset of the central directory.
The second matters for entries such as shards, which can be large and whose final size may not be known when they are written.
Using the extra fields for all entries adds tens of bytes per entry: in the libraries we tested, 20 bytes per local file header and up to about 32 further bytes per central directory record.
We consider this worthwhile relative to the gigabyte to terabyte scale at which OME-Zarr datasets are expected to grow; for example, a 1 TB dataset stored as about 1,000 shards of 1 GB carries well under 100 kB of such overhead.

We agree that guidance for common languages is needed, and will include library-specific notes for writing ZIP64 in the implementation guidance of the RFC.
Libraries differ in which of these pieces they can write: for example, of the libraries we tested, Python's `zipfile` can write the extra fields in the local headers but not the end records for small archives, while the JDK's zip file system can write the end records (through an undocumented property) but not the extra fields.
We have therefore kept this as a SHOULD instead of downgrading it to a MAY, and have expanded the corresponding text in the _Proposal_ section to state what is required by the ZIP format itself, what we recommend, and why it is not a requirement.

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

### Minor comments and questions

> The specification section has the line "Amend the specification with the following section:" Is this a reference to the section that immediately follows?

Yes; we have reworded the sentence to make this explicit.

> The example json for the ome attribute in the zip archive has an extraneous comma on the line `"jsonFirst": true`

Thank you, the comma has been removed.

> The phrase: "These disadvantages were considered to be outweighed by other aspects (see Proposal section)." looks like it should be unindented so it applies to the whole "Drawbacks" section

Agreed; the sentence is now a separate paragraph after the list of drawbacks.

> Is the limitation (imposed by enforcing single-file archival) of the total size of an OME-Zarr image that can be represented as a single file this way a Drawback?

Yes, in practice: the limits of ZIP64 are far beyond practical sizes, but very large single files are difficult to handle (file system and object store limits, transfer and copy costs).
We have added this to the drawbacks, noting that directory-backed or object store-backed OME-Zarr remains an option for very large datasets.

> Compatibility: Is it fully compatible with zarr < 0.5?

No: see the response above ("Versions of OME Zarr"); the RFC applies only to OME-Zarr v0.5 and later.

> Is the example dataset at https://static.webknossos.org/misc/6001240.ozx supposed to be accessible by viewers? [...] Access to fetch [...] has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.

> [Neuroglancer link] with acl to public and CORS to allow all origins works.

Thank you for testing this.
The server that hosts the example dataset does not currently send CORS headers, which we confirmed, so browser-based viewers other than the linked Neuroglancer demo cannot read it directly; this is a property of the hosting, not of the format.
We have added a note on the required CORS configuration to the _Tutorials and Examples_ section, and added a set of 51 example datasets, hosted in a bucket that allows cross-origin requests, to the _Implementation_ section. [Open: consider re-hosting `6001240.ozx` with CORS enabled.]

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

Like other archive formats with a trailing index, a ZIP archive that is corrupted or only partially transferred is hard to use directly, because the central directory is at the end of the file.
Since every entry is also preceded by a local file header, and since this RFC requires that entries are stored without ZIP-level compression and that no data precedes the first local file header, recovery tools can often scan forward through the archive and rebuild the central directory.
This is not guaranteed, e.g. when entries were written with data descriptors (sizes recorded after the entry data).
Recovery is not a primary goal of this RFC, and we do not specify additional recovery mechanisms.

##### Performance

> Is fast performance a goal of this format? If so, how fast? [...] Can that be achieved well with a single-file zipped OME-Zarr structure? How does it compare to unzipped OME-Zarr?

Yes: we expect reading OME-Zarr zip files to be comparable to reading other Zarr stores when readers use range requests and the recommendations are followed.
We have rewritten the _Performance_ section of the RFC to summarize the most relevant findings of the existing external evaluations (see also the response to comment 5 below), and to state this expectation.
We have measured central directory costs on synthetic archives (scripts in `scripts/rfc9/`): parsing time and central directory size grow linearly with the number of entries (about 0.6 s and 59 MB at one million entries with a minimal parser), lookups after indexing are independent of the number of entries, and with `jsonFirst` the hierarchy is discovered in time independent of the number of entries. These measurements are local and do not include network latency. [Open: summarize these results in the RFC; random-access chunk read latency compared to an unzipped store has not been measured.]
The _Drawbacks_ section already describes the logic that readers may need to optimize lookups (parsing the central directory into a hash table or sorting it).

##### Use cases

This is addressed in the response to comment 3 (see the corresponding section of this document).

#### Generalizing `jsonFirst`

> For future-proofing, I suggest generalizing this field beyond only a boolean. It would make sense as a field defining the nature of the tree structure. Something like "treeStructure": "levelOrder" (i.e. breadth first).

`jsonFirst` asserts a property of the set of records, not of their order: every `zarr.json` record precedes every other record in the central directory, not counting directory entries.
This is exactly the property a reader needs in order to stop parsing the central directory early and still know that the structure of the hierarchy is complete; it does not depend on any particular order among the `zarr.json` records.
A separate flag for breadth-first ordering was proposed during the drafting of the RFC and later removed for this reason.
We have clarified this in the RFC (including that `jsonFirst` does not assert breadth-first order), and we have added an example.
Since the keys under `ome` are strictly defined by the specification and are extended by future versions of it, an additional parameter describing the order of the `zarr.json` records can be added later should a use case arise.

#### Explicitly constrain ZIP options

> Expand the "OME-Zarr zip files" section of the specification to more fully enumerate which ZIP features are permitted vs. forbidden (e.g., encryption not permitted; no extra fields beyond specified metadata; mutation permitted or not)

Thank you for the concrete proposal.
We have gone through the suggested list item by item:

- **Encryption (any method):** adopted for the ZIP archive level. OME-Zarr zip files MUST NOT use ZIP encryption. Codec-level encryption is out of scope for this RFC.
- **Multi-volume or split archives:** already a MUST NOT in the draft (requirement 4).
- **Compression at ZIP level:** adopted. OME-Zarr zip files MUST NOT use ZIP-level compression (STORE method only), since compression is expected to be performed by Zarr-level codecs. This may be relaxed in the future, e.g. for `zarr.json` documents.
- **Archive comments beyond the specified `ome` JSON:** partly adopted. The archive comment is encoded as JSON precisely so that future versions of the specification can add parameters to it, for versioning and for parameterized features. The keys under the top-level `ome` attribute are strictly defined by the specification, and we have clarified this in the RFC. Other top-level keys are permitted, to allow composition with other specifications. The `jsonFirst` parameter lets viewers discover the hierarchy without parsing the whole central directory, similar to the tree views of HDF5 viewers such as HDFView or h5web; we have expanded its description in the RFC.
- **Extra fields containing non-Zarr data:** adopted in a refined form. ZIP extra fields do not carry Zarr data but per-entry metadata (e.g. the ZIP64 information, timestamps, Unix permissions), so we prohibit all extra fields except an enumerated set of standard metadata fields (ZIP64 extended information, extended timestamp, Info-ZIP New Unix). This lets us extend the list in future versions of the specification.
- **Self-extracting ZIP code:** adopted in a general form. OME-Zarr zip files MUST NOT contain any data before the first local file header, which prohibits self-extracting archives and other prepended stubs. This may be revisited in a future version of the specification.
- **Mutation:** permitted, with the caveats described in the _Access and mutability_ subsection of the RFC (see also the response above).

Regarding "Validators MUST reject ozx files violating these constraints": we agree that the MUST requirements need to be testable. We have added a requirement that a validator for OME-Zarr zip files MUST report a violation of any of the MUST or MUST NOT requirements. [ozx-tck](https://github.com/clbarnes/ozx-tck) is intended for this purpose.

#### Provide implementation guidance

> **Provide implementation guidance**: Show concrete examples of constructing ozx files, demonstrate performance characteristics, and highlight some languages/libraries that meet the spec's constraints.

We have added implementation guidance as scripts and notes that accompany the RFC (the `scripts/rfc9/` directory):

- **Examples of constructing .ozx files**: self-contained writers for Python with the standard library `zipfile` and with the third-party package `stream-zip`, and for Java with the JDK alone and with Apache Commons Compress. They are tested against the checker described below.
- **A checker** (`check_ozx.py`) that tests the MUST requirements of this RFC on a local file or a URL using range requests. It found no violations of the checked requirements in the 51 example datasets listed in Zipglancer.
- **Library notes** (`scripts/rfc9/examples/README.md`) that state, for each library tested, which of the requirements and recommendations it can meet and where it falls short, e.g. the lack of control over the central directory order, the lack of an archive comment in the JDK zip file system and in `stream-zip`, and differences in how libraries write ZIP64 (see also the ZIP64 recommendations in the RFC). The RFC lists the libraries and tools that support OME-Zarr zip files in its _Implementation_ section.
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
It lists the intended use cases (file-centric desktop workflows, transport and archival, viewing and exploring remote data with range requests, and exchange between Zarr implementations), points to current demonstrations of them, and lists what OME-Zarr zip files are not intended for (many concurrent writers or frequent in-place modification, streaming, archive-level encryption, and very large numbers of entries without sharding).
The format is primarily write-once read-many, as suggested, but not exclusively: some mutability is expected (see the response to the comment on mutability above).

The demonstrations we found in the record of this RFC and elsewhere are:

- the Neuroglancer view of the example dataset `6001240.ozx` (see the _Tutorials and Examples_ section);
- WEBKNOSSOS, which can read OME-Zarr zip files remotely and uploaded ones ([pull request](https://github.com/scalableminds/webknossos/pull/9738), merged);
- Zipglancer, a web-based explorer for ZIP and OME-Zarr zip files using range requests, which provides links to 51 example OME-Zarr zip files in a public bucket (all of which pass the checks for the requirements of this RFC that `check_ozx.py` can perform on the central directory, and which all use the `jsonFirst` ordering);
- converters and test tools: the Rust converter `ozx` and the validator `ozx-tck`.

This list is not exhaustive. [Open: coordinate with the co-author assigned to the use case documentation and add any further demonstrations.]

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
[Open: confirm with co-authors that the order of the entries themselves is no longer recommended.]
Regarding duplicate `zarr.json` entries observed when appending: the RFC now recommends that writers remove duplicate records from the central directory at the end of a writing session. [Open: define which record a reader should use if duplicates remain.]

#### ZIP disadvantage in performance

> Compared to a directory store, file content is not necessarily stored page-aligned. [...] This point could be added under the drawback section in the RFC.

Thank you for sharing this experience.
We have added the lack of page alignment, the resulting read-modify-write cycles for unbuffered page-aligned I/O, and the mitigation of allocating a separate page per local file header (at the cost of additional space, acceptable with sharding) to the _Drawbacks_ section of the RFC.
Note that padding by means of ZIP extra fields is not permitted by this RFC (see the requirement on extra fields); leaving unused space between entries, as described in the comment, remains possible.

#### Sharding constraints

> [...] partial writes are impractical unless the final size of each shard is known in advance. As a result, it is often not recommended to shard an axis that is acquired sequentially. [...] This point could be added under the drawback section in the RFC. The recommendation to SHOULD use sharding depends of course on the chunk size, the expected number of chunks and the codec pipeline.

Agreed; we have added this to the _Drawbacks_ section and made the dependence of the sharding recommendation on these factors explicit.

#### CRC/hash requirement

> ZIP requires CRC32 for file entries, which is useful for integrity verification, but it burdens implementations, especially for partial writes and reads. [...] clarifying recommended strategies (e.g., validating at shard or chunk granularity and deferring CRC checks for in-flight writes) would help implementers. [...] This point could be added under the drawback section in the RFC.

We have added this to the _Drawbacks_ section, including the observation on hardware support for CRC-32C but not for the CRC-32 used by ZIP, which we attribute to the comment.
The RFC now states that implementations may validate at shard or chunk granularity and defer CRC-32 checks for entries that are still being written. [Open: confirm with co-authors that this is meant as an allowance and not as a recommendation.]

#### ZIP disadvantage when updating

> [...] we allowed in-place updates as long as the size does not grow beyond the existing space. As an example, we added capacity (padding) to allow in-place updates of metadata. (Similar to `tiffcomment` or `tiffset` on tiff files). [...] We just wanted to mention here how this could be mitigated in certain cases.

Thank you; we have added this mitigation to the _Drawbacks_ section.
Padding by means of ZIP extra fields is not permitted by this RFC (see the requirement on extra fields); reserving unused space between entries remains possible.

#### Split archives

> [...] we see use cases where archive-level splitting would be beneficial, particularly from a user-experience perspective. However, we acknowledge that this adds complexity to implementations, and support this decision.

Thank you for your support of this decision.
Splitting remains a possible future extension; see the _Future possibilities_ section.

#### Thumbnails

> Applications might benefit from pre-rendered thumbnails. [...] it might be a question if this should be a topic to be addressed by zipped OME-Zarr separately or if this is out of scope for this RFC.

We consider this out of scope for this RFC and applicable to OME-Zarr in general; see our response to the same question in review 1.

#### Recommend specific implementations

> In general, the specification could recommend in the end using specific implementations over standard ZIP writers so that end users can create compatible .ozx files and avoid interoperability issues.

We agree with the intent: the RFC states that OME-Zarr zip files are expected to be produced primarily by OME-Zarr-aware tooling rather than by generic ZIP tools, and the _Implementation_ section lists the tools and libraries that support them.
We did not make a recommendation for specific implementations part of the specification, since this would tie the specification to the current state of the ecosystem; instead, the implementation guidance that accompanies the RFC (scripts, a checker, and notes on the capabilities of several libraries) helps implementers and users to create and verify compatible files.

## Comment 5

Response to [comment 5](https://ngff.openmicroscopy.org/rfc/9/comments/5/index.html) by Anna Kreshuk, Dominik Kutra, and Dominik Kutra, Ilastik.

### Significant comments and questions

#### Clarify performance expectations

> For an RFC to a public standard, in our opinion, at least the most relevant information from the external source should be reflected in the text.

Agreed.
The _Performance_ section now summarizes the most relevant findings of the two external sources in the text, including the key numbers and the observation that the reader implementation matters more than the use of ZIP itself, and notes that those evaluations concern geospatial rather than microscopy data.

> "When creating OME-Zarr zip files, the following RECOMMENDATIONS ensure that reading from OME-Zarr zip files is similarly performant as reading from other storage formats:"

We adopted a variant of this phrasing for the list of recommendations, stating that they are intended to ensure reading performance similar to other storage formats.
We did not adopt the stronger formulation as a normative requirement on writers, since writers cannot fully control reader implementations; instead, the _Performance_ section advises writers to verify read performance for their data. [Open: confirm with co-authors whether to make this a SHOULD.]

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
