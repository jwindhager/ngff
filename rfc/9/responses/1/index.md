# RFC-9: Response 1

(rfcs:rfc9:response1)=

## Review 1

Response to [review 1](https://ngff.openmicroscopy.org/rfc/9/reviews/1/index.html) by Pete Bankhead, University of Edinburgh.

### Minor comments and questions

#### Image preview

> The 'preview' aspect makes it tempting to want to embed a thumbnail, which could be supported by some applications or operating system plugins. Should this be explicitly forbidden / discouraged / encouraged in a standard way?

We agree that thumbnails are particularly relevant for single-file use cases.
However, we believe that thumbnail support applies to OME-Zarr in general and should therefore be proposed separately.

## Review 2

Response to [review 2](https://ngff.openmicroscopy.org/rfc/9/reviews/2/index.html) by Kola Babalola and Matthew Hartley, BioImage Archive, EMBL-EBI.

## Review 3

Response to [review 3](https://ngff.openmicroscopy.org/rfc/9/reviews/3/index.html) by Curtis Rueden, University of Wisconsin-Madison.

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
