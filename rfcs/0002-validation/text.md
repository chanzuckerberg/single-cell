# Validation and Conversion of Uploaded Datasets

## [Optional] Subtitle of my design

**Authors:** [Marcus Kinsella](mkinsella@chanzuckerberg.com)

**Approvers:** [Friendly Tech Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly PM Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly Tech Person2](mailto:some.nice.person@chanzuckerberg.com), [Girish Patangay](mailto:girish.patangay@chanzuckerberg.com), [Sara Gerber](mailto:sara.gerber@chanzuckerberg.com) 

## tl;dr 

We want uploaded datasets to follow our schema and be converted into multiple file formats. This describes how users are
able to create datasets that follow the schema and the process and infrastructure we use to verify that and reformat the
dataset.

## Problem Statement | Background

There isn't a broad community consensus around how single cell datasets should be stored and annotated. This
makes it harder to integrate these datasets, which is bad because that's a main aim of our single cell work. So, we have
a simple schema that we want all datasets to adhere to. If all the datasets we store at least follow this schema, then
our data corpus will be usable for data integration research and the production of atlases.

The problem is that getting a dataset to follow the schema isn't trivial. The schema is pretty lightweight, but there's
a bunch of data munging, relabeling of columns, etc. that's at best a pain and at worst beyond the capabilities of some
submitters. So we want some tooling to help with that, and we also want to make sure that before we actually add a
dataset to the portal/cellxgene we verify that it follows the schema.

Finally, single cell datasets come in different formats that are associated with different toolchains. We want to
support as many of those toolchains as we can, so we're going to convert datasets into different formats. It seems to
make sense to do this right after validation is complete since we're already parsing the original file anyway, and
there's not really a good reason to delay converting.


## Product Requirements

The key requirement here is that users can prepare their own datasets without intervention from CZI. This goes along
with all the other work to support self-submission: file upload, collection create, private/public publication.


## Detailed Design | Architecture | Implementation

There are two main phases for the preparation of a dataset. In the first, the submitter is working with their dataset
locally, preparing it so it follows the schema and is in an accepted file format. We have command line tools to help
with that are part of cellxgene. The second phase occurs after the dataset file is uploaded, and that it the phase we
are mainly concerned about here.

Through some upload process (described in an RFC elsewhere) a dataset file ends up in S3. Initially, that file is
expected be in the AnnData HDF5 format, but eventually other formats will be supported as well. Once that happens, we
need to validate and convert that file, which occurs in a number of steps:

1. Localize the uploaded file to the local storage of some instance (HDF5 doesn't like remote files usually).
2. Run the validation tool on the file. This is the same tool that's available to users via the cellxgene CLI, so this
   is really just verifying that they ran it. This should only take a couple of seconds and use little memory.
3. Update the database with the result of validation. If it fails, stop here.
4. Convert the uploaded file to the Loom format.
5. Convert the uploaded file to a Seurat object and save it.
6. Convert the uploaded file to a SingleCellExperiment object and save it.
7. Copy the converted files back to S3 and update the database with their locations.

Steps 4, 5, and 6 can occur in parallel. They're also slower and sometimes use a lot of memory. They're performed using
community tools like [sceasy](https://github.com/cellgeni/sceasy), which can have tricky dependencies. Generally, the
step whose time impacts user experience is 2. Users will want to know right away if their file is accepted, but they're
more willing to wait for conversion to complete.

The simplest approach is to run all the steps in a single container. The memory requirements for steps 4, 5, and 6 are
much higher than the other steps, but all the other steps are very fast. And several of the steps share complex
dependencies, so running them together can keep things simpler.

This can be implemented with a pretty standard Fargate/ECS setup that's triggered by events in the uploaded files
bucket. We can create an image with all the dependencies for the various R and python scripts and include that in a task
that gets executed after every new upload.


### Monitoring and error reporting


## Alternatives

Other teams at CZI have used Airflow to orchestrate ETL tasks. Airflow comes with a nicer interface and more flexbility
around triggering tasks but more overhead in terms of infrastructure to maintain.

## References
