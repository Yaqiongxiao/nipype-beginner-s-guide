.. important::

    This guide hasn't been updated since January 2017 and is based on an older version of Nipype. The code in this guide is not tested against newer Nipype versions and might not work anymore. For a newer, more up to date and better introduction to Nipype, please check out the the `Nipype Tutorial <https://miykael.github.io/nipype_tutorial/>`_.

====================
Prepare your Dataset
====================

We can't start to learn how to use Nipype before we don't have any data to work on. The fact that you are interested in using Nipype implies that you probably already have access to an MRI dataset. You are of course free to do your first steps with Nipype directly on your own dataset. But if you're new to the topic or want to be sure that you get the same results like I do, I recommend to use a tutorial dataset which we can find over on `OpenfMRI <https://openfmri.org/>`_.


Download the tutorial dataset
=============================

.. image:: _static/logo/logoOpenfmri.png
   :width: 120pt
   :align: left

`OpenfMRI <https://openfmri.org/>`_ is an awesome new online database where you can upload ans share your fMRI dataset or download datasets from other researchers. Such an open approach to science allows you and other researchers to profit from each other and helps the field as a whole to progress faster. A list of all datasets available on OpenfMRI can be found on `openfmri.org <https://openfmri.org/dataset/>`_.


Download the dataset from OpenfMRI
----------------------------------

I chose to use the dataset `DS102: Flanker task (event-related) <https://openfmri.org/dataset/ds000102>`_ as the tutorial dataset for this beginner’s guide because the dataset is complete, in good quality, the experiment is representative for most fMRI experiments and the dataset as a whole is with its 1.8GB not too big to download. To download the dataset go to the button of the `DS102: Flanker task (event-related) <https://openfmri.org/dataset/ds000102>`_ page and click on the link: `Raw data on AWS <https://openfmri.s3.amazonaws.com/tarballs/ds102_raw.tgz>`_.

.. note::

    If you are on a Linux system, you can also download the tutorial dataset directly to your ``Downloads`` folder with the following command:

    .. code-block:: sh

        wget https://openfmri.s3.amazonaws.com/tarballs/ds102_raw.tgz ~/Downloads


Unpack and prepare the folder structure
---------------------------------------

The file you've just downloaded contains amongst others one anatomical and two functional MRI scans and the behavioral response and onsets of stimuli during those functional scans. The whole dataset consists of 26 subjects. Information about gender and age of each subject is stored in the `demographics.txt` file, also found in downloaded file. To reduce the total time of computation, this tutorial will only analyze the first 10 subjects of the whole dataset. Now let's get ready.

What we want is an experiment folder called ``nipype_tutorial`` that contains a folder called ``data`` where we save the raw data of the first 10 subjects into. In the end, the structure should be something like this

.. code-block:: sh

    nipype_tutorial
    |-- data
        |-- demographics.txt
        |-- sub001
        |   |-- behavdata_run001.txt
        |   |-- behavdata_run002.txt
        |   |-- onset_run001_cond001.txt
        |   |-- onset_run001_cond002.txt
        |   |-- onset_run001_cond003.txt
        |   |-- onset_run001_cond004.txt
        |   |-- onset_run002_cond001.txt
        |   |-- onset_run002_cond002.txt
        |   |-- onset_run002_cond003.txt
        |   |-- onset_run002_cond004.txt
        |   |-- run001.nii.gz
        |   |-- run002.nii.gz
        |   |-- struct.nii.gz
        |-- sub0..
        |-- sub010
            |-- behav...
            |-- onset_...
            |-- run...
            |-- struct.nii.gz


This can either be done manually or with the following code:

.. code-block:: sh
    :linenos:

    # Specify important variables
    ZIP_FILE=~/Downloads/ds102_raw.tgz   #location of download file
    TUTORIAL_DIR=~/nipype_tutorial       #location of experiment folder
    TMP_DIR=$TUTORIAL_DIR/tmp            #location of temporary folder
    DATA_DIR=$TUTORIAL_DIR/data          #location of data folder

    # Unzip ds102 dataset into TMP_DIR
    mkdir -p $TMP_DIR
    tar -zxvf $ZIP_FILE -C $TMP_DIR

    # Copy data of first ten subjects into DATA_DIR
    for id in $(seq -w 1 10)
    do
        echo "Creating dataset for subject: sub0$id"
        mkdir -p $DATA_DIR/sub0$id
        cp $TMP_DIR/ds102/sub0$id/anatomy/highres001.nii.gz \
           $DATA_DIR/sub0$id/struct.nii.gz

        for session in run001 run002
        do
            cp $TMP_DIR/ds102/sub0$id/BOLD/task001_$session/bold.nii.gz \
               $DATA_DIR/sub0$id/$session.nii.gz
            cp $TMP_DIR/ds102/sub0$id/behav/task001_$session/behavdata.txt \
               $DATA_DIR/sub0$id/behavdata_$session.txt

            for con_id in {1..4}
            do
                cp $TMP_DIR/ds102/sub0$id/model/model001/onsets/task001_$session/cond00$con_id.txt \
                   $DATA_DIR/sub0$id/onset_${session}_cond00$con_id.txt
            done
        done

        echo "sub0$id done."
    done

    # Copy information about demographics, conditions and tasks into DATA_DIR
    cp $TMP_DIR/ds102/demographics.txt $DATA_DIR/demographics.txt
    cp $TMP_DIR/ds102/models/model001/* $DATA_DIR/.

    # Delete the temporary folder
    rm -rf $TMP_DIR

.. hint::

    You can download this code as a script here: `tutorial_1_create_dataset.sh <https://github.com/miykael/nipype-beginner-s-guide/blob/master/scripts/tutorial_1_create_dataset.sh>`_


Acquire scan and experiment parameters
--------------------------------------

One of the most important things when analyzing any data is to know your data. What does it consist of, how was it recorded, what are its characteristics and most importantly, what does it look like. The `DS102: Flanker task (event-related) <https://openfmri.org/dataset/ds000102>`_ page helps us already to answer many of those parameters:

Scan parameters
...............


    * Magnetic field: 3T [Tesla]
    * Head coil: Siemens standard, which probably means 32-channel
    * Information about **functional** acquisition
        * Acquisition type: contiguous echo planar imaging (EPI)
        * Number of volumes per session: 146
        * Number of slices per volume: 40
        * Slice order: unknown
        * Repetition time (TR): 2000ms
        * Field of view (FOV): 64x64
        * Voxel size: 3x3x4mm

    * Information about **anatomical** acquisition
        * Acquisition type: magnetization prepared gradient echo sequence (MPRAGE)
        * Number of slices: 176
        * Repetition time (TR): 2500ms
        * Field of view (FOV): 256mm

Experiment parameters
.....................

    * **Task**: On each trial, participants used one of two buttons on a response pad to indicate the direction of a central arrow in an array of 5 arrows. In *congruent* trials the flanking arrows pointed in the same direction as the central arrow (e.g., < < < < <), while in more demanding *incongruent* trials the flanking arrows pointed in the opposite direction (e.g., < < > < <).
    * **Condition**: congruent and incongruent
    * **Scan sessions**: Subjects performed two 5-minute blocks, each containing 12 congruent and 12 incongruent trials, presented in a pseudo-random order.

So what do we know?
...................

    * We know that we have two functional scans per subjects, in our case called `run001` and `run002`. Each functional scan represents 5min of scan time and consists of 146 volumes. Each of those volume consists 40 slices (with a thickness of 4mm) and each slice consists of 64x64 voxel with the size of 3x3mm. The TR of each volume is 2000ms.
    * So far we don't know what the slice order of the functional acquisition is. But a closer look at the provided references tells us that the data was acquired with an *interleaved* slice to slice order. The fact that it is not stated if it is ascending or descending interleaved means that the acquisition is most certainly ascending.
    * We know that we have one anatomical scan per subject, in our case called `struct` and that this anatomical scan consists of 176 slices with each having a FOV of 256mm. This hints to an isometric voxel resolution of 1x1x1mm.
    * We know the task of the experiment and that it consists of two conditions, *congruent* and *incongruent*. With each condition being presented 12 times per session.
    * We know from the onset file found in the subject folder what the actual onset of the two conditions are, but we have no clear information about the duration of each stimulation. The `description of the design on OpenfMRI.org <https://openfmri.org/dataset/ds000102>`_ tells us that the inter-trial interval varies between 8 and 14s with a mean of 12s. A closer look at the references tells us that the subjects first see a fixation cross for 500ms, followed by the congruent or incongruent condition (i.e. the arrows) for 1500ms, followed by a blank screen for 8000-12000ms.


Check the data
--------------

It is always important to look at your data and verify that it actually is recorded the way it should be. To take a look at the data, I usually use FreeSurfer's ``freeview``. For example, if you want to load the anatomical scan of all ten subjects use the following code:

.. code-block:: sh

    freeview -v ~/nipype_tutorial/data/sub00*/struct.nii.gz

To verify the information about the scan parameters we can use ``fslinfo``. ``fslinfo`` allows us to read the header information of a NIfTI file and therefore get information about voxel resolution and TR. For example, reading the header of the anatomical scan of subject2 with the command ``fslinfo ~/nipype_tutorial/data/sub002/struct.nii.gz`` gives us following output:

.. code-block:: sh

    data_type      INT16
    dim1           176
    dim2           256
    dim3           256
    dim4           1
    datatype       4
    pixdim1        1.000000
    pixdim2        1.000000
    pixdim3        1.000000
    pixdim4        0.000000
    cal_max        0.0000
    cal_min        0.0000
    file_type      NIFTI-1+

This output tells us, that the anatomical volume consists of 256x256x176 voxels with each having 1x1x1mm resolution.

Using the same command on a functional scan of subject 2 gives as following output:

.. code-block:: sh

    data_type      INT16
    dim1           64
    dim2           64
    dim3           40
    dim4           146
    datatype       4
    pixdim1        3.000000
    pixdim2        3.000000
    pixdim3        4.000000
    pixdim4        2000.000000
    cal_max        0.0000
    cal_min        0.0000
    file_type      NIFTI-1+

This output tells us that this functional scan consists of 146 volumes, of which each consists of 64x64x40 voxels with a resolution of 3x3x4mm. `pixdim4` gives us additionally information about the TR of this functional scan.

.. note::

    Just as a side note: The `data_type` of a NIfTI file tells you the amount of bits used to store the value of each voxel. **INT16** stands for *signed short* (16 bits/voxel), **INT32** stands for *signed int* (32 bits/voxel) and **FLOAT64** stands for *float* (64 bits/voxel). The more bits used for storing a voxel value the bigger the whole NIfTI file.

    If you want to change the data type of your image, either use Nipype's function ``ChangeDataType`` found in the ``nipype.interfaces.fsl.maths`` package (read more `here <http://nipype.readthedocs.io/en/latest/interfaces/generated/nipype.interfaces.fsl.maths.html#changedatatype>`_) or use ``fslmaths`` directly to change the data_type to INT32 with the following command:

    .. code-block:: sh

        fslmaths input.nii output.nii -odt int


For those who use their own dataset
-----------------------------------

If you want to use your own dataset, make sure that you know the following parameters:

    * Number of volumes, number of slices per volume, slice order and TR of the functional scan.
    * Number of conditions during a session, as well as onset and duration of stimulation during each condition.

.. important::

    Make sure that the layout of your data is similar to the one stated above, so that further code is also applicable for your case.


Make the dataset ready for Nipype
=================================

Convert your data into NIfTI format
-----------------------------------

You don't have to do this step if you're using the tutorial dataset. But chances are that you soon want to analyze your own recorded dataset. And most often, the images coming directly from the scanner are not in the common ``NIfTI`` format, but rather in a scanner specific format (e.g. ``DICOM``, ``PAR/REC``, etc.). This means you first have to convert your data from this specific scanner format to the standard NIfTI format.

Probably the most common scanner format is DICOM. Therefore, the following section will cover how you can convert your files from DICOM to NIfTI. There are many different tools that you can use to convert your files. For example, if you like to have a nice GUI to convert your files, use `MRICron <http://www.mccauslandcenter.sc.edu/mricro/mricron/>`_'s `MRIConvert <http://lcni.uoregon.edu/~jolinda/MRIConvert/>`_. But for this Beginner's Guide we will use FreeSurfer's ``mri_convert`` function, as it is rather easy to use and doesn't require many steps.

But first, as always, be aware of your folder structure. So let's assume that we've stored our dicoms in a folder called ``raw_data`` and that the folder structure looks something like this:

.. code-block:: none

    raw_dicom
    |-- sub001
    |   |-- t1w_3d_MPRAGE
    |   |   |-- 00001.dcm
    |   |   |-- ...
    |   |   |-- 00176.dcm
    |   |-- fmri_run1_long
    |   |   |-- 00001.dcm
    |   |   |-- ...
    |   |   |-- 00240.dcm
    |   |-- fmri_run2_long
    |       |-- ...
    |-- sub0..
    |-- sub010

This means, that we have one folder per subject with each containing another folder, one for the structural T1 weighted image and 2 for the functional T2 weighted images. The conversion of the dicom files in those folders is rather easy. If you use FreeSurfer's ``mri_convert`` function, the command is as as follows: ``mri_convert <in volume> <out volume>``. You have to replace ``<in volume>`` by the actual path to any one dicom file in the folder and ``<out volume>`` with the name for your outputfile.

So, to accomplish this with some few terminal command, we first have to tell the system the path and names of the folders that we later want to feed to the ``mri_convert`` function. This is done by the following variables (line 1 to 6). If this is done, we only have to run the loop (line 8 to 17) to actually run ``mri_convert`` for each subject and each scanner image.

.. code-block:: sh
    :linenos:

    TUTORIAL_DIR=~/nipype_tutorial     # location of experiment folder
    RAW_DIR=$TUTORIAL_DIR/raw_dicom    # location of raw data folder
    T1_FOLDER=t1w_3d_MPRAGE            # dicom folder containing anatomical scan
    FUNC_FOLDER1=fmri_run1_long        # dicom folder containing 1st     functional scan
    FUNC_FOLDER2=fmri_run2_long        # dicom folder containing 2nd functional scan
    DATA_DIR=$TUTORIAL_DIR/data        # location of output folder

    for id in $(seq -w 1 10)
    do
        mkdir -p $DATA_DIR/sub0$id
        mri_convert $RAW_DIR/sub0$id/$T1_FOLDER/00001.dcm    $DATA_DIR/sub0$id/struct.nii.gz
        mri_convert $RAW_DIR/sub0$id/$FUNC_FOLDER1/00001.dcm $DATA_DIR/sub0$id/run001.nii.gz
        mri_convert $RAW_DIR/sub0$id/$FUNC_FOLDER2/00001.dcm $DATA_DIR/sub0$id/run002.nii.gz
    done


Run FreeSurfer's recon-all
--------------------------

Not mandatory but highly recommended is to run FreeSurfer's ``recon-all`` process on the anatomical scans of your subject. ``recon-all`` is FreeSurfer's cortical reconstruction process that automatically creates a `parcellation of cortical <https://surfer.nmr.mgh.harvard.edu/fswiki/CorticalParcellation>`_ and a `segmentation of subcortical <http://freesurfer.net/fswiki/SubcorticalSegmentation>`_ regions. A more detailed description about the ``recon-all`` process can be found on the `official homepage <http://surfer.nmr.mgh.harvard.edu/fswiki/recon-all>`_.

As I said, you don't have to use FreeSurfer's ``recon-all`` process, but you want to! Because many of FreeSurfer's other algorithms require the output of ``recon-all``. The only negative point about ``recon-all`` is that it takes rather long to process a single subject. My average times are between 12-24h, but it is also possible that the process takes up to 40h. All of it depends on the system you are using. So far, ``recon-all`` can't be run in parallel. Luckily, if you have an 8 core processor with enough memory, you should be able to process 8 subjects in parallel.



Run recon-all on the tutorial dataset (terminal version)
........................................................

The code to run ``recon-all`` on a single subject is rather simple, i.e. ``recon-all -all -subjid sub001``. The only thing that you need to keep in mind is to tell your system the path to the freesurfer folder by specifying the variable ``SUBJECTS_DIR`` and that each subject you want to run the process on has a according anatomical scan in this freesurfer folder under ``SUBJECTS_DIR``.

To run ``recon-all`` on the 10 subjects of the tutorial dataset you can run the following code:

.. code-block:: bash
    :linenos:

    # Specify important variables
    export TUTORIAL_DIR=~/nipype_tutorial         #location of experiment folder
    export DATA_DIR=$TUTORIAL_DIR/data            #location of data folder
    export SUBJECTS_DIR=$TUTORIAL_DIR/freesurfer  #location of freesurfer folder

    for id in $(seq -w 1 10)
    do
        echo "working on sub0$id"
        mkdir -p $SUBJECTS_DIR/sub0$id/mri/orig
        mri_convert $DATA_DIR/sub0$id/struct.nii.gz \
                    $SUBJECTS_DIR/sub0$id/mri/orig/001.mgz
        recon-all -all -subjid sub0$id
        echo "sub0$id finished"
    done


This code will run the subjects in sequential order. If you want to process the 10 subjects in (manual) parallel order, delete line 12 - ``recon-all -all -subjid sub0$id`` - from the code above, run it and than run the following code, each line in its own terminal:

    .. code-block:: sh

            export SUBJECTS_DIR=~/nipype_tutorial/freesurfer; recon-all -all -subjid sub001
            export SUBJECTS_DIR=~/nipype_tutorial/freesurfer; recon-all -all -subjid sub002
            ...
            export SUBJECTS_DIR=~/nipype_tutorial/freesurfer; recon-all -all -subjid sub010


.. note::

    If your MRI data was recorded on a 3T scanner, I highly recommend to use the ``-nuintensitycor-3T`` flag on the ``recon-all`` command, e.g. ``recon-all -all -subjid sub0$id -nuintensitycor-3T``. This flag was created specifically for 3T scans and `improves the brain segmentation accuracy by optimizing non-uniformity correction using N3 <http://web.mysites.ntu.edu.sg/zvitali/publications/documents/N3_NI.pdf>`_.

.. hint::

    You can download this code as a script here: `tutorial_2_recon_shell.sh <https://github.com/miykael/nipype-beginner-s-guide/blob/master/scripts/tutorial_2_recon_shell.sh>`_


Run recon-all on the tutorial dataset (Nipype version)
......................................................

If you run ``recon-all`` only by itself, I recommend you to use the terminal version shown above. But of course, you can also create a pipeline and use Nipype to do the same steps. This might be better if you want to make better use of the parallelization implemented in Nipype or if you want to put ``recon-all`` in a bigger workflow.

I won't explain to much how this workflow actually works, as the structure and creation of a common pipeline is covered in more detail in the next section. But to use Nipype to run FreeSurfer's ``recon-all`` process do as follows:

.. code-block:: py
    :linenos:

    # Import modules
    import os
    from os.path import join as opj
    from nipype.interfaces.freesurfer import ReconAll
    from nipype.interfaces.utility import IdentityInterface
    from nipype.pipeline.engine import Workflow, Node

    # Specify important variables
    experiment_dir = '~/nipype_tutorial'             # location of experiment folder
    data_dir = opj(experiment_dir, 'data')  # location of data folder
    fs_folder = opj(experiment_dir, 'freesurfer')  # location of freesurfer folder
    subject_list = ['sub001', 'sub002', 'sub003',
                    'sub004', 'sub005', 'sub006',
                    'sub007', 'sub008', 'sub009',
                    'sub010']                        # subject identifier
    T1_identifier = 'struct.nii.gz'                  # Name of T1-weighted image

    # Create the output folder - FreeSurfer can only run if this folder exists
    os.system('mkdir -p %s'%fs_folder)

    # Create the pipeline that runs the recon-all command
    reconflow = Workflow(name="reconflow")
    reconflow.base_dir = opj(experiment_dir, 'workingdir_reconflow')

    # Some magical stuff happens here (not important for now)
    infosource = Node(IdentityInterface(fields=['subject_id']),
                      name="infosource")
    infosource.iterables = ('subject_id', subject_list)

    # This node represents the actual recon-all command
    reconall = Node(ReconAll(directive='all',
                             #flags='-nuintensitycor-3T',
                             subjects_dir=fs_folder),
                    name="reconall")

    # This function returns for each subject the path to struct.nii.gz
    def pathfinder(subject, foldername, filename):
        from os.path import join as opj
        struct_path = opj(foldername, subject, filename)
        return struct_path

    # This section connects all the nodes of the pipeline to each other
    reconflow.connect([(infosource, reconall, [('subject_id', 'subject_id')]),
                       (infosource, reconall, [(('subject_id', pathfinder,
                                                 data_dir, T1_identifier),
                                                'T1_files')]),
                       ])

    # This command runs the recon-all pipeline in parallel (using 8 cores)
    reconflow.run('MultiProc', plugin_args={'n_procs': 8})


After this script has run, all important outputs will be stored directly under ``~/nipype_tutorial/freesurfer``. But the running of the ``reconflow`` pipeline also created some temporary files. As defined by the script above, those files were stored under ``~/nipype_tutorial/workingdir_reconflow``. Now that the script has run you can delete this folder again. Either do this manually, use the shell command ``rm -rf ~/nipype_tutorial/workingdir_reconflow`` or add the following lines to the end of the python script above:

.. code-block:: py

    # Delete all temporary files stored under the 'workingdir_reconflow' folder
    os.system('rm -rf %s'%reconflow.base_dir)

.. note::

    In the code above, if we don't create the ``freesurfer`` output folder on line 19, we would get following error:

    .. code-block:: py

        TraitError: The 'subjects_dir' trait of a ReconAllInputSpec instance must be an existing
        directory name, but a value of '~/nipype_tutorial/freesurfer' <type 'str'> was specified.

    Also, if your data was recorded on a 3T scanner and you want to use the mentioned ``-nuintensitycor-3T`` flag, just uncomment line 32, i.e. delete the ``#`` sign before ``flags='-nuintensitycor-3T'`` on line 32.

.. hint::

    You can download this code as a script here: `tutorial_2_recon_python.py <https://github.com/miykael/nipype-beginner-s-guide/blob/master/scripts/tutorial_2_recon_python.py>`_


Resulting Folder Structure
==========================

After we've prepared our data and run the ``recon-all`` process the folder structure of our experiment folder should look as follows:

.. code-block:: sh

    nipype_tutorial
    |-- rawdata (optional)
    |-- data
    |   |-- sub001
    |   |-- sub0..
    |   |-- sub010
    |-- freesurfer
        |-- sub001
        |-- sub0..
        |-- sub010

