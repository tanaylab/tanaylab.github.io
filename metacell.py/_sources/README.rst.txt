Metacell - Single-cell RNA Sequencing Analysis
==============================================

The metacell package implements the metacell algorithm for single-cell RNA sequencing (scRNA-seq)
data analysis.

The `original <https://www.biorxiv.org/content/10.1101/437665v1>`_ metacell algorithm was
implemented in R. In contrast, the Python package described here implements an `improved
<https://todo/>` version, which uses improved internal algorithms and supports much larger data sets
(millions of cells).

Metacell Analysis
-----------------

Naively, scRNA_seq data is a set of cell profiles, where for each one, for each gene, we get a count
of the mRNA molecules that existed in the cell for that gene. This serves as an indicator of how
"expressed" or "active" the gene is.

As in any real world technology, the raw data may suffer from technical artifacts (counting the
molecules of two cells in one profile, counting the molecules from a ruptured cells, counting only
the molecules from the cell nucleus, etc.). This requires pruning the raw data to exclude such
artifacts. TODO: Create view should consider these issues.

The current technology scRNA-seq data is also very sparse (typically <10%, sometimes even <5% of the
RNA molecules are counted). This introduces large sampling variance on top of the original signal,
which itself contains significant inherent biological noise.

Analyzing scRNA-seq data therefore requires processing the profiles in bulk. Classically, this has
been done by directly clustering the cells using various methods.

In contrast, the metacell approach groups together profiles of the "same" biological state into
groups with the *minimal* number of profiles (~100) needed for computing robust statistics (in
particular, mean gene expression). Each such group is a single "metacell".

By summing profiles together, each metacell greatly reduces the sampling variance, and provides a
more robust estimation of some transcription state. In particular, a metacell is not a cell type
(multiple metacells may belong to the same type), and is not a parametric model of the cell state.

The metacells should therefore be further analyzed using additional methods to classify cell types,
detect cell trajectories and/or lineage, build parametric models for cell behavior, etc. Using
metacells as input for such analysis techniques should benefit both from the more robust, less noisy
input; and from the (~100-fold) reduction in the number of profiles to analyze.

An obvious technique is to recursively group the metacells into larger groups, and investigate the
resulting clustering hierarchy. Methods for further analysis of the data may choose to build upon
this readily available hierarchy (which is also provided by this package).

Usage
-----

Installation
............

Given Python version 3.6 and above, run ``pip install --user metacell`` (or, ``sudo pip install
metacell`` if you have ``sudo`` privileges). This will install this package and two scripts:
``metacell_flow`` and ``metacell_call``.

The metacell package is built using the `DynaMake <https://pypi.org/project/dynamake/>`_ package.
The ``metacell_flow`` script is simply the ``dynamake`` script, pre-loaded with the metacell build
steps, and ``metacell_call`` is simply the ``dynacall`` script, pre-loaded with the metacell
computation functions.

Importing
.........

To use the package, create a directory somewhere with a large amount of available space (for ~2M
cells, metacell required up to 200GB of disk space). If using a servers cluster, on some shared file
system, and provide a DynaMake configuration file with an appropriate ``run_prefix`` for
distributing jobs across the cluster; see the DynaMake documentation for details.

Create a ``raw`` sub-directory and download the raw_data_ into it. Then, run ``metacell_flow
import`` to import the gene and cell data into the fast-access format used by the metacell package.
This is unfortunately a slow process (in particular, the delaning step). Luckily, you only need to
do this once.

Analysis
........

TODO: Analyze the raw data to determine exclusions.

TODO: Create a view for analysis.

Once a view has been created, run ``metacell_flow --view=name group``. This will result in a
``views/name/profiles.group.npy`` file containing the group index of each of the profiles, and a
``views/name/groups`` directory with the fast-access data for these computed groups (and groups of
groups, etc., as needed).

TODO: Investigate the results.

Directory Structure
-------------------

The provided automated workflow uses the following directory structure. The specifics of each
directory type are provided further below.

``ROOT``
    A directory which is typically named after the data source (``PBMC``, ``MOCA``, ``HCA``, etc.).
    Will contain:

    ``raw``
        A directory containing the raw data from the data source. Should contain one or more:

        ``name``
            A directory containing 10X_ or MOCA_ data.

        ``name.h5``
            An HDF file containing HCA_ data.

    ``std``
        The first processing step will convert the ``raw`` data to the standard input format. The
        result will contain:

        ``name``
            A directory in the standard (STD_) format ready to be delaned.

        Creating the standard format directory is pretty fast. It typically requires either a simple
        copy or applying something like ``sed`` to fix minor formatting issues. For HCA files, it is
        just a matter of dumping the data into a textual format.

    ``delaned``
        The next processing step will split the input into distinct lanes. The result will contain:

        ``name``
            A directory for each of the standard format directories. This would be either a standard
            format directory, or will contain:

            ``lane``
                A directory in the standard format containing just the data for a single lane.

        When data contains multiple lanes, delaning is a painfully slow process as it needs to split
        the matrix market file, processing one line at a time (without any parallelization). It
        would have been much easier if the raw data was already split into lanes.

    ``genes``
        A directory containing fast access per-gene data (GENES_) extracted from the ``delaned``
        data.

        The extraction process first verifies this data is identical for all the ``delaned``
        directories. Then one of these directories is used to actually extract the data.
        This is very fast, as there are only ~30,000 genes to deal with.

    ``cells``
        A directory containing fast access per-gene-per-profile (COLLECTION_) data. Will contain:

        ``name``
            A directory for each of the ``delaned`` directories. This would either be a
            simple profiles batch (BATCH_), or a collection combining:

            ``lane``
                A profiles batch directory containing the data for a single lane.

        Conversion of the (delaned) textual data to the fast-access format isn't very fast, but this
        is offset by being able to run this conversion in parallel for each lane. It is therefore
        much less painful than delaning. Once conversion is done, the import process is complete and
        does not need to be repeated.

    ``views``
        This is where the metacells package performs its analysis, recursively grouping the cells
        into metacells. Should contain:

        ``name``
            Some view profiles container (VIEW_) based on the full ``cells`` profiles collection, to
            group. Typically such a view will exclude profiles with too-low total UMIs count, or
            that suffer from other technical issues (e.g. doublets). Genes may also be excluded.

            The main function of the metacell package is to recursively compute the groups for
            such a view (that is, extend it to become a GROUPED_ directory).

.. _raw_data:

Raw Data
........

The formats which can be used as the raw input for metacell processing:

.. _10X:

**10X**
    A directory containing `10X Genomics <https://www.10xgenomics.com/>`_ data. Should contain the
    files:

    ``matrix.mtx``
        Matrix market UMIs data.

    ``barcodes.tsv``
        The barcode of each profile (possibly with a ``-lane`` suffix).

    ``genes.tsv``
        The ensembl ID and name of each gene.

.. _MOCA:

**MOCA**
    A directory containing `Mouse Organogenesis Cell Atlas
    <http://atlas.gs.washington.edu/mouse-rna/>`_ data. Should contain the files:

    ``gene_count.txt``
        Matrix market UMIs data.

    ``cell_annotate.csv``
        Per-cell data (see list of expected fields in ``metacell/storage/moca_format.yaml``).

    ``gene_annotate.csv``
        Per-gene data (see list of expected fields in ``metacell/storage/moca_format.yaml``).

.. _HCA:

**HCA**
    A file containing `Human Cell Atlas <https://www.humancellatlas.org/>`_ in HDF format with a
    `.h5`` suffix. Must contain a single top-level key with the sub-keys ``barcodes``, ``indptr``,
    ``genes``, ``indices`` and ``data``.

These formats are converted to a standard input format:

.. _STD:

**STD**
    A directory containing data in a standard metacell input format. Should contain the files:

    ``umis.mtx``
        Matrix market UMIs data.

    ``profiles.csv``
        Per-profile data.

    ``genes.csv``
        Per-gene data.

    ``format.yaml``
        A description of the fields of the CSV files.

Fast Access Data
................

Formats of data allowing fast access during the metacell processing.

.. _GENES:

**GENES**
    A directory containing per-gene data, accessed using the
    :py:class:`metacell.storage.genes.Genes` class. Should contain the files:

    ``genes.yaml``
        General meta-data describing this set of genes. Should contain the keys ``organism``,
        ``genes_count`` and ``uuid``, which must be equal to the ``md5sum`` of the
        ``genes.name.txt`` file. This UUID uniquely identifies this set of genes.

    ``genes.*.txt``
        Per-gene string data. These should include ``genes.name.txt`` containing the
        (sorted, all-upper-case) gene names, and ``genes.ensembl.txt`` containing the
        `ensembl <https://www.ensembl.org>`_ identifier of each gene.

    ``genes.*.npy``
        Per-gene numeric data, if any.

.. _CONTAINER:

**CONTAINER**
    Either a **BATCH**, **COLLECTION**, or **VIEW** directory, providing access to
    per-gene-per-profile data for some genes and profiles using the
    :py:class:`metacell.storage.profiles.ProfilesContainer` class.

.. _BATCH:

**BATCH**
    A directory containing per-gene-per-profile data for some closely related profiles (a single
    batch), accessed using the :py:class:`metacell.storage.profiles.ProfilesBatch` class. Barcodes
    can be safely assumed to be unique within each batch, but not between multiple batches. When
    investigating batch effects, this is the smallest group of profiles for which these effects can
    be assumed to be the same. Should contain the files:

    ``profiles_batch.yaml``
        General meta-data describing this batch of profiles. Should contain the keys:

        ``profiles_count``, ``genes_count``
            The number of profiles, and number of genes per profiles.
        ``profiles_kind``
            The kind of profiles (``cells``, ``metacells``, ``1_zones``, ``2_zones``, etc.).
        ``genes_uuid``
            The UUID of the genes per profile.
        ``uuid``
            Should be equal to the ``md5sum`` of the ``matrix.umis.mmm`` or ``matrix.umis.npy``
            file. This UUID uniquely identifies this profiles batch.

    ``matrix.*.mmm``, ``matrix.*.npy``
        Per-gene-per-profile data. The ``mmm`` format is a metacell-specific compressed sparse data
        format accessed using the :py:class:`metacell.matrices.MemoryMappedMatrix` class. Each batch
        should contain ``matrix.umis.mmm`` or ``matrix.umis.npy``, but additional
        per-gene-per-profile data may exist.

    ``profiles.*.txt``
        Per-profile string data. Common data includes:

        ``profiles.barcode.txt``
            The unique (in this batch) per-profile barcode (without any lane suffix).

        ``profiles.name.txt``
            The unique (in this batch) per-profile string name, if any.

        ``profiles.batch.txt``, ``profiles.lane.txt``
            The name of the batch and the name of the lane (if any) per profile. These are the same
            for all profiles in the batch, but having these files is useful when collecting multiple
            batches to a single data set (see below).

    ``profiles.*.npy``
        Per-profile numeric data. Common data includes:

        ``profiles.total_umis.npy``, ``profiles.total_squared_umis.npy``
            The sum of the genes UMIs and the sum of the square of the genes UMIs for each profile.

    ``genes.*.txt``, ``genes.*.npy``
        Per-gene string and numeric data, specific for this profiles batch, if any. Common data
        includes:

        ``genes.total_umis.npy``, ``genes.total_squared_umis.npy``
            The sum of the profiles UMIs and the sum of the square of the profiles UMIs for each
            gene.

.. _VIEW:

**VIEW**
    A directory containing a possibly restricted view of another profiles container accessed using
    the :py:class:`metacell.storage.profiles.ProfilesView` class. Should contain the files:

    ``profiles_view.yaml``
        General meta-data describing the profiles view. Should contain the keys:

        ``base_uuid``
            The UUID of the base profiles container this is a view of.

        ``base_path``
            The path of the base profiles container this is a view of. If this is a relative path,
            it is relative to the view directory.

        ``profiles_kind``
            Identical to the value of the base profiles container.

        ``profiles_count``, ``genes_count``
            The number of profiles and genes in the restricted view. These are less than or equal to
            the values of the base profiles container.

        ``genes_uuid``
            The unique identifier of the genes subset. This will be identical to the UUID of the
            genes of the base container if there is no ``included_genes.npy`` file, that is,
            if the view uses all the base container's genes.

        ``uuid``
            If the view is identical to the base profiles container (that is, acts as a link to it),
            then should be the same as the UUID of the base profiles container. Otherwise, is based
            on the combination of the UUID of the base profiles container, and the bytes of the
            ``included_profiles.npy`` and/or ``included_genes.npy`` files.

    ``included_profiles.npy``
        If exist, contain the sorted indices of the base container profiles which are included in
        this view. Otherwise, the view contains all the base container profiles.

    ``included_genes.npy``
        If exist, contain the sorted indices of the base container genes which are included in this
        view. Otherwise, the view contains all the base container genes.

    ``matrix.*.mmm``, ``matrix.*.npy``
        Per-gene-per-profile data, specific for the specific view, if any.

    ``profiles.*.txt``, ``profiles.*.npy``
        Per-profile data, specific for the specific view, if any.

    ``genes.*.txt``, ``genes.*.npy``
        Per-gene data, specific for the specific view, if any.

.. _COLLECTION:

**COLLECTION**
    A directory combining a collection of other profiles container directories into a single unified
    profiles container, accessed using the :py:class:`metacell.storage.profiles.ProfilesCollection`
    class.
    Should contain the files:

    ``profiles_collection.yaml``
        General meta-data describing the combined container. Should contain the keys:

        ``uuid``
            Should be equal to the ``md5sum`` of all the ``uuid`` bytes of the contained batches.
            This UUID uniquely identifies the profiles collection.

        ``profiles_count``
            The total number of profiles in the collection.

        ``profiles_kind``, ``genes_uuid``, ``genes_count``
            Identical to the values in each of the combined profile batches.

        ``combined``
            A list of mapping, one per combined profile container directories. Should contain the
            keys:

            ``name``
                The name of a sub-directory of the collection directory, containing the profiles
                container which is combined into the overall collection.

            ``uuid``
                The UUID identifying the combined profiles container.

            ``profiles_count``
                The number of profiles in the combined profiles container.

    ``genes.*.txt``, ``genes.*.npy``
        Per-gene data, specific for the overall collection, if any.

    ``name``
        A profiles container sub-directory which is combined into the collection.

Grouped Data
............

The primary function of the metacell package is to group profiles together, in particular, to group
cell profiles into metacells.

.. _GROUPED:

**GROUPED**
    A profiles container directory can become "grouped" if it also contains the files:

    ``profiles.group.npy``
        The group index for each of the container's profiles. A negative value indicates the profile
        belongs to no group (is an "outlier").

    ``groups``
        A **GROUPS** profiles container for the computed groups. The per-profile-per-gene UMIs count
        in this container will be the sum of the per-profile-per-gene of each of the grouped
        profiles contained in the group profile.

        A groups profiles container has identical format to a normal profiles batch directory,
        except that its metadata file contains a ``grouped_profiles`` key with the (relative) path
        to the grouped profiles container (typically ``..``), and a ``grouped_uuid`` key with the
        UUID of the grouped profiles container.

        The ``profiles_kind`` of the groupes container should be based on the grouped container
        profiles kind. In particular, groups of ``cells`` are called ``metacells``. Such groups are
        expected to contain cells of the "same" state.

        When grouping a large number of profiles, the ``groups`` directory might itself be grouped.
        This recursion continues as long as the total number of profiles is "large". The
        ``profiles_kind`` of higher level groups is named ``1_zones``, ``2_zones`` etc. Such groups
        contain profiles with a "similar", but not necessarily the "same" state.

    ``outliers``
        A profiles view containing just the outlier profiles (which are not included in any
        of the groups).

    ``tmp``
        Temporary files used to compute the groupings, but need not be used once it has been
        computed. The contents of this directory depends on whether we are grouping a few
        profiles or many profiles.

        If grouping a small number of profiles:

        ``selected``
            A view of the profiles container, including only the selected ("feature") genes.

        If grouping a large number of profiles, we can't directly compute the groups, since the
        basic algorithm has a complexity of O(N^2). Computation therefore must works through a
        divide-and-conquer algorithm. This requires different temporary files:

        ``phase-1``
            A directory containing the second phase's files.

            ``pile-index``
                A profiles view of a random subset of a restricted number of to-be-grouped profiles.
                To help in sorting file names, the indices are zero-padded so all have the same
                width (e.g. ``00, .., 10, 11, ...``). Each such pile is independently (directly)
                grouped.

            ``outliers-pile``
                This directory is a profiles view containing all profiles that were marked as an
                outlier in some indexed pile. It is also independently grouped. Unlike the indexed
                piles, this view may contain a large number of profiles so grouping it may require
                an intermediate step of its own.

                Rare "types" do not have enough profiles in each of the piles to form groups, so
                will be marked as outliers. In this outliers-only view, there might be enough of
                these rare profiles to form a proper group.

        ``grouped``
            A view of the grouped profiles container, with ``p.group_index.npy`` containing the
            combined group indices from all the piles.

        ``groups``
            The intermediate grouping combines the groups from all the phase-1 piles, as well as the
            groups from the phase-1 outliers. This is the less-accurate grouping we will improve
            upon. To allow doing so, we compute the groups-of-groups, that is, we compute
            ``p.group_index.npy``. This may require recursion.

        ``outliers``
            A profiles view containing just the profiles which were marked as outliers when
            grouping the phase-1 outliers described above. That is, all the profiles which
            we tried to but failed to group in the first phase.

        ``phase-2``
            A directory containing the second phase's files. This is similar to the first phase's
            structure, with the following differences:

            - The profiles contained in each pile are not chosen randomly. Instead, they are the
              profiles contained in one of the computed intermediate groups-of-groups.

            - The outliers pile contains the phase-1 outliers in addition to the outliers of the
              phase-2 indexed piles. This allows very rare profile "types" a second chance at being
              grouped.

            - There are no ``grouped`` or ``groups`` directories; the results are written directly
              to the original (parent) grouped profiles container. These final groups are
              recursively grouped as long as there is a large number of profiles, as described
              above.
