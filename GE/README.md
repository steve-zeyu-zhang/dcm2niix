## About

dcm2niix attempts to convert GE DICOM format images to NIfTI. The current generation DICOM files generated by GE equipment is quite impoverished relative to other vendors. Therefore, the amount of information dcm2niix is able to extract is relatively limited. Hopefully, in the future GE will provide more details that are critical for brain scientists.

## Arterial Spin Labeling

As noted by David Shin (GE), the GE product ASL sequence sequence produces two 3D volumes. The Perfusion Weighted (PW) first pass acquires ASL tag/control spirals in interleaved fashion over many volumes (TRs), and does the subtraction and averaging in k-space. Therefore, the result is a single 3D volume. The second pass acquires a Proton Density (PD) reference volume. The PW and PD images can be combined offline to generate quantified Cerebral Blood Flow(CBF) map. [Stanford](https://cni.stanford.edu/wiki/Data_Processing) includes useful notes on this sequence. Note that Number of Excitations (NEX) is needed for CBF quantification. The sequence specific details are listed in the table. 

| DICOM Tag | Pass 1 (PW)                        | Pass 2 (PD)                        |
|-----------|------------------------------------|------------------------------------|
| 0043,10A3 | PSEUDOCONTINUOUS                   | CONTINUOUS                         |
| 0043,10A4 | 3D pulsed continuous ASL technique | 3D continuous ASL technique        |
| 0043,10A5 | Label Duration (ms)                | Label Duration (ms)                |
| 0018,0082 | Post Label Delay (ms)              | NA                                 |
| 0027,1060 | Number of Points per Arm           | Number of Points per Arm           |
| 0027,1061 | Number of Arms                     | Number of Arms                     |
| 0027,1062 | Number of Excitations              | Number of Excitations              |

The GE product ASL sequence is optimized for clinical diagnosis with emphasis on 3D acquisition and multi-shot interleaving for high spatial resolution. For 4D type acquisition (time resolved single-shot acquisition), GE researchers can leverage the [multi-band ASL/BOLD sequence](https://journals.plos.org/plosone/article/authors?id=10.1371/journal.pone.0190427).

## Diffusion Tensor Notes

The [NA-MIC Wiki](https://www.na-mic.org/wiki/NAMIC_Wiki:DTI:DICOM_for_DWI_and_DTI#Private_vendor:_GE) provides a nice description of the GE diffusion tags. In brief, the B-value is stored as the first element in the array of 0043,1039. The DICOM elements 0019,10bb, 0019,10bc and 0019,10bd provide the gradient direction relative to the frequency, phase and slice. As noted by Jaemin Shin (GE), the GE convention for reported diffusion gradient direction has always been in “MR physics” logical coordinate, i.e Freq (X), Phase (Y), Slice (Z). Note that this is neither “with reference to the scanner bore” (like Siemens or Philips) nor “with reference to the imaging plane” (as expected by FSL tools). This is the main source of confusion. This explains why the dcm2niix function geCorrectBvecs() checks whether the DICOM tag In-plane Phase Encoding Direction (0018,1312) is 'ROW' or 'COL'. In addition, it will generate the warning 'reorienting for ROW phase-encoding untested' if you acquire DTI data with the phase encoding in the ROW direction. If you do test this feature, please report your findings as a Github issue. Assuming you have COL phase encoding, dcm2niix should provide [FSL format](http://justinblaber.org/brief-introduction-to-dwmri/) [bvec files](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FDT/FAQ#What_conventions_do_the_bvecs_use.3F).

## Slice Timing

Knowing the relative timing of the acquisition for each 2D slice in a 3D volume is useful for [slice time correction](https://www.mccauslandcenter.sc.edu/crnl/tools/stc) of both fMRI and DTI data. Unfortunately, current GE software does not provide a consistent way to record this.

[Some sequences](https://afni.nimh.nih.gov/afni/community/board/read.php?1,154006) encode the RTIA Timer (0021,105E) element. For example, [this DV24 dataset](https://github.com/nikadon/cc-dcm2bids-wrapper/tree/master/dicom-qa-examples/ge-mr750-slice-timing) includes timing data, while [this DV26 dataset does not](https://github.com/neurolabusc/dcm_qa_nih). Be aware that different versions of GE software appear to use different units for 0021,105E. The DV24 example is reported in seconds, while [14.0 uses 1/10000 seconds](https://github.com/rordenlab/dcm2niix/issues/286). An example of the latter format can be found [here](https://www.nitrc.org/plugins/mwiki/index.php/dcm2nii:MainPage#Archival_MRI). Even with the sequences that do encode the RTIA Timer, there is some debate regarding the accuracy of this element. In the example listed, the slice times are clearly wrong in the first volume. Therefore, dcm2niix always estimates slice times based on the 2nd volume in a time series.

In general, fMRI acquired using GE product sequence (PSD) “epi” with the multiphase option will store slice timing in the Trigger Time (DICOM 0018,1060) element. In contrast, the popular PSD “epiRT” (BrainWave RT, fMRI/DTI package provided by Medical Numerics) does not save this tag (though in some cases it saves the RTIA Timer). Examples are [available](https://www.nitrc.org/plugins/mwiki/index.php/dcm2nii:MainPage#Slice_timing_correction) for both the “epiRT” and “epi” sequences. The [“epi2” is a special case](https://github.com/rordenlab/dcm2niix/issues/635).

The “epiRT” sequences also allow the user to specify the `Group Delay`, which is 0 msec by default. Increasing this value will create a pause at the end of each volume, and this value is recorded in the DICOM header(as 0043,107C, reported in seconds). This option can be used for sparse designs, where one wants a pause after each value. Be aware that the `RepetitionTime` (0018,0080) reported in this header omits the group delay. So a study with a TR of 2000ms and a Group Delay of 55ms will report the values (0018,0080 = 2000, 0043,107C = 0.055), while the actual sampling rate will be 2055ms.  This is unintuitive, the TR with respect to tissue contrast is 2055ms, not the reported 2000.

If neither Trigger Time (DICOM 0018,1060) or RTIA Timer (0021,105E) store slice timing information, a final option is to decode the GE Protocol Data Block as described below. At best, this block only reports whether the acquisition was interleaved or sequential. As long as one assumes the acquisition was continuous (with no temporal gap between volumes, e.g. sparse images) on can use this value, the number of slices in the volume and the repetition time to infer slice times.

Due to these various methods, recent releases of dcm2niix read the Protocol Data Block to determine the multi-bandFactor, number of samples, sampling rate (TR), interleaved or sequential slices, group delay and software version. This information is used to estimate the slice timing directly, without requiring the previously described tags. The [dcm_qa_ge](https://github.com/neurolabusc/dcm_qa_ge) provides validation images and more details. Note that this slice timing approach does not support GE's diffusion weighted imaging sequences.

## User Define Data GE (0043,102A)

This private element of the DICOM header is used to determine the phase encoding polarity. Specifically, we need to know the "Ky traversal direction" (top-down, or bottom up) and the phase encoding polarity. Unfortunately, this data is stored in a complicated, proprietary structure, that has changed with different releases of GE software. [Click here to see the definition for this structure](https://github.com/ScottHaileRobertson/GE-MRI-Tools/blob/master/GePackage/%2BGE/%2BPfile/%2BHeader/%2BRDB15/rdbm.h).

## Multi-Echo EPI Sequences

Current GE software (DV26.0_R03_1831.b) running research multi-echo sequences create invalid DICOM images. The required public [EchoTime (0018,0081)](https://dicom.innolitics.com/ciods/mr-image/mr-image/00180081) attribute lists the shortest echo time for the series, rather than the actual echo time for the given DICOM image. The public tag [EchoNumber (0018,0086)](https://dicom.innolitics.com/ciods/mr-image/mr-image/00180086) reports `1` for all echoes. These limitations in GE's DICOM images disrupt dcm2niix's image conversion. Hopefully future product sequences will generate valid DICOM data. In the meantime, [issue 359](https://github.com/rordenlab/dcm2niix/issues/359) provides a kludge for image conversion.

## Image Interpolation

Some sequences allow the user to interpolate images in plane (e.g. saving a 2D 64x64 EPI image as 128x128) or between slices (e.g. saving a 126 slice T1-weighted image as 252 images). The resulting files require much more disk space, add no new information, are slower to process and can [disrupt some tools](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrdegibbs.html). Users are strongly discouraged from interpolating raw data. However, dcm2niix should correctly detect this interpolation, resolving apparent discrepancies between tags (0020,1002; 0021,104F; 0054,0081). [Issue 355](https://github.com/rordenlab/dcm2niix/issues/355) provides details.

## Total Readout Time

One often wants to determine [echo spacing, bandwidth](https://support.brainvoyager.com/brainvoyager/functional-analysis-preparation/29-pre-processing/78-epi-distortion-correction-echo-spacing-and-bandwidth) and total read-out time for EPI data so they can be undistorted. Specifically, we are interested in FSL's definition of total read-out time, which may differ from the actual read-out time. FSL expects “the time from the middle of the first echo to the middle of the last echo, as it would have been had partial k-space not been used”. So total read-out time is influenced by parallel acceleration factor, bandwidth, number of EPI lines, but not partial Fourier. For GE data we can use the Acquisition Matrix (0018,1310) in the phase-encoding direction, the in-plane acceleration ASSET R factor (the reciprocal of this is stored as the first element of 0043,1083) and the Effective Echo Spacing (0043,102C). While GE does not tell us the [partial Fourier fraction](https://bids-specification.readthedocs.io/en/stable/04-modality-specific-files/01-magnetic-resonance-imaging-data.html), is does reveal if it is present with the ScanOptions (0018,1022) reporting [PFF](http://dicomlookup.com/lookup.asp?sw=Ttable&q=C.8-4) (in my experience, GE does not populate [(0018,9081)](http://dicomlookup.com/lookup.asp?sw=Tnumber&q=(0018,9081))). While partial Fourier does not impact FSL's totalReadoutTime directly, it can interact with the number of lines acquired when combined with parallel imaging (the `Round_factor` 2 (Full Fourier) or 4 (Partial Fourier)).

The formula for FSL's definition of TotalReadoutTime (in seconds) is:

```
TotalReadoutTime = ( ( ceil ((1/Round_factor) * PE_AcquisitionMatrix / Asset_R_factor ) * Round_factor) - 1 ] * EchoSpacing * 0.000001
EffectiveEchoSpacing = TotalReadoutTime/ (reconMatrixPE - 1)
```

Consider an example:

```
(0018,1310) US 128\0\0\128                                 #   8, 4 AcquisitionMatrix
(0018,0022) CS [SAT_GEMS\MP_GEMS\EPI_GEMS\ACC_GEMS\PFF\FS] #  42, 6 ScanOptions
(0043,102c) SS 636                                         #   2, 1 EchoSpacing
(0043,1083) DS [0.666667\1]                                #  10, 2 Acceleration
```

From this we can derive:

```
ASSET= 1.5 PE_AcquisitionMatrix= 128 EchoSpacing= 636 Round_Factor= 4 TotalReadoutTime= 0.055332
```

## Image Acceleration

The BIDS [ParallelReductionFactorInPlane](https://bids-specification.readthedocs.io/en/stable/04-modality-specific-files/01-magnetic-resonance-imaging-data.html#in-plane-spatial-encoding) can be determined from the reciprocal of the first element of the private tag `Asset R Factors` (0043,1083). The first value is for acceleration in the phase direction equivalent to the public tag 0018,9069), the second for the slice direction (equivalent to the public tag 0018,9155). For 2D EPI scans, the second value will always be 1, whereas for 3D acquisitions [acceleration can take place in both the phase-encoding and the slice-encoding directions](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4459721/). For example, a scan using a acceleration factor of 1.5 would be reported as `(0043,1083) DS [0.666667\1]`.

The BIDS [MultibandAccelerationFactor](https://bids-specification.readthedocs.io/en/stable/04-modality-specific-files/01-magnetic-resonance-imaging-data.html#rf-contrast) can be determined from the private tag `Multiband Parameters` (0043,10B6). This is an array with at least three values, the first is the Multiband (aka HyperBand) factor, the second is the Slice FOV Shift Factor, and the final is the Calibration method. For example, a scan using a multiband factor of 2 could be reported as `(0043,10b6) LO [2\4\19\\\\]`. 

## Missing Information

While GE DICOMs will report if the image uses partial Fourier, it will not reveal the partial Fourier fraction.

## Phase-Encoding Polarity

All EPI scans have spatial distortion, particularly those with longer readout times. Tools like [FSL topup](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/topup/TopupUsersGuide) can leverage data where two spin-echo images are acquired that are identical except for using opposite phase-encoding polarity (e.g. one uses A>P, the other P>A). Each image is distorted with the same magnitude, but in the opposite direction.  GE's Rx27 software version and later populate the [Rectilinear Phase Encode Reordering (0018,9034)](http://dicomlookup.com/lookup.asp?sw=Tnumber&q=(0018,9034)) tag which for EPI is set to either LINEAR or REVERSE_LINEAR. This should be given precedence to similar values in [User Define Data GE (0043,102A)](https://github.com/rordenlab/dcm2niix/issues/674).

## GE Protocol Data Block

In addition to the public DICOM tags, previous versions of dcm2niix attempted to decode the proprietary GE Protocol Data Block (0025,101B). This is essentially a [GZip format](http://www.onicos.com/staff/iz/formats/gzip.html) file embedded inside the DICOM header. Unfortunately, this data seems to be [unreliable](https://github.com/rordenlab/dcm2niix/issues/163) and therefore this strategy is not used anymore. The notes below regarding the usage of this data block are provided for historical purposes.

 - The VIEWORDER tag is used to set the polarity of the BIDS tag PhaseEncodingDirection, with VIEWORDER of 1 suggesting bottom up phase encoding. Unfortunately, users can separately reverse the phase encoding direction making this tag unreliable. Thankfully, recent scanners provide 0018,9034.
 - The SLICEORDER tag could be used to set the SliceTiming for the BIDS tag PhaseEncodingDirection, with a SLICEORDER of 1 suggesting interleaved acquisition.
 - There are reports that newer versions of GE equipement (e.g. DISCOVERY MR750 / 24\MX\MR Software release:DV24.0_R01_1344.a) are now storing an [XML](https://groups.google.com/forum/#!msg/comp.protocols.dicom/mxnCkv8A-i4/W_uc6SxLwHQJ) file within the Protocol Data Block (compressed). In theory this might also provide useful information.
 
## Complex Image Component

Most vendors store the complex image component (magnitude, phase, real or imaginary) in the [Complex Image Component (0008,9208)](http://dicom.nema.org/medical/dicom/2016e/output/chtml/part03/sect_C.8.13.3.html) tag or in the [Image Type (0008,0008)](http://incenter.medical.philips.com/doclib/enc/fetch/2000/4504/577242/577256/588723/5144873/5144488/5144982/DICOM_Conformance_Statement_Intera_R7%2c_R8_and_R9.pdf%3fnodeid%3d5147977%26vernum%3d-2) tag. GE stores this information as a signed short of the Private Image Type(0043,102F) tag. The values 0, 1, 2, 3 correspond to magnitude, phase, real, and imaginary (respectively).

## Detecting Anatomical Localizers

Anatomical localizers (e.g. scout images) are quick-and-dirty scans used to position subsequent slower but higher quality images. These scans are typically discarded in subsequent analyses. The dcm2niix argument `-i y` will ignore these scans. For GE, these sequences are detected based on the SeriesPlane (0019,1017) tag, which is of type [SS](http://dicom.nema.org/dicom/2013/output/chtml/part05/sect_6.2.html) and can report the values 2 (Axial), 4 (Sagittal), 8 (Coronal), 16 (Oblique) or 256 (3plane). The 3plane value is consistent with an anatomical localizer. 

## Sample Datasets

 - [A validation dataset for dcm2niix commits](https://github.com/neurolabusc/dcm_qa_nih).
 - [Slice Timing and Phase Encoding examples](https://github.com/jannikadon/cc-dcm2bids-wrapper/tree/main/dicom-qa-examples)
 - [Slice timing validation](https://github.com/neurolabusc/dcm_qa_stc) for different varieties of GE EPI sequences.
 - [Examples of phase encoding polarity, slice timing and diffusion gradients](https://github.com/neurolabusc/dcm_qa_ge).
 - The dcm2niix [wiki](https://www.nitrc.org/plugins/mwiki/index.php/dcm2nii:MainPage) includes examples of diffusion data, slice timing, and other variations.

