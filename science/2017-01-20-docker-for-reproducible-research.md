# Docker in reproducible research and hackatons 

## First time I git an interest in the subject
 - First through reproducible research as a general discipline, and the potential of docker and vagrant/packer for reproducible computing and data science
 - The latest occurence was the [talk python to me \#91 podcast](https://talkpython.fm/episodes/show/91/top-10-data-science-stories-of-2016) when they talked about the [DREAM challenge](https://www.synapse.org/#!Synapse:syn4224222/wiki/401743).

## What about it
Digging further in the DREAM challenge website and finding this [presentation in particular](http://fr.slideshare.net/Docker/docker-in-open-science-data-analysis-challenges-by-bruce-hoff), we can clearly see the use of docker as a data "provider" for untrained sets of data for the challenge. This extract from the presentation of Bruce hoff is excellent to explain the different issues that this approach solves.


>Typically in predictive data analysis challenges, participants are provided a dataset and asked to make predictions. Participants include with their prediction the scripts/code used to produce it. Challenge administrators validate the winning model by reconstructing and running the source code.

`_translator's note_``The following part is essential and can be applied to several fields` 
>Often data cannot be provided to participants directly, e.g. due to data sensitivity (data may be from living human subjects) or data size (tens of terabytes). Further, predictions must be reproducible from the code provided by particpants. Containerization is an excellent solution to these problems: Rather than providing the data to the participants, we ask the participants to provided a Dockerized "trainable" model. We run the both the training and validation phases of machine learning and guarantee reproducibility 'for free'.

>We use the Docker tool suite to spin up and run servers in the cloud to process the queue of submitted containers, each essentially a batch job. This fleet can be scaled to match the level of activity in the challenge. We have used Docker successfully in our 2015 ALS Stratification Challenge and our 2015 Somatic Mutation Calling Tumour Heterogeneity (SMC-HET) Challenge, and are starting an implementation for our 2016 Digitial Mammography Challenge. 
