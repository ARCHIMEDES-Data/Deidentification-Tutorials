---
jupyter:
  jupytext:
    main_language: python
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.19.3
  kernelspec:
    display_name: Python 3
    name: python3
---

<!-- #region id="Erl8u-UXtf3N" -->
# **De-Identification of DICOM Data - Jupyter Notebook Tutorial**

**Authors:**
Nadia Farrag, Ph.D. &
Jefferson Casimir

---

This script-style Jupyter notebook introduces researchers to Python-based tools for reducing identifiability in DICOM data. It demonstrates common technical approaches used in de-identification workflows, with examples informed in part by the Information and Privacy Commissioner of Ontario (IPC) & HIPAA Safe Harbor concepts for identifiers commonly removed from health data.

*This tutorial assumes a basic working knowledge of Python coding and familiarity with running Jupyter notebooks.*



It allows users to:



*   Use either a sample or custom DICOM file
*   Select specific DICOM tags and apply de-identification actions such as clearing, masking, generalization, pseudonymization, or suppression
*   Filter and batch process files from a folder



Select specific DICOM tags and apply de-identification actions such as clearing, masking, generalization, pseudonymization, or suppression
Filter and batch process files from a folder

--------------

## **IMPORTANT DISCLAIMER:**

This tutorial is provided for **educational and research workflow support only**. It does not provide legal advice, and **it does not by itself determine whether data are de-identified, anonymized, or otherwise compliant under any specific law or institutional policy**. Users are responsible for ensuring that their data handling practices meet the requirements of the laws, regulations, ethics approvals, data sharing agreements, and institutional policies applicable to their project.

***For Canadian users:***

Methods commonly used to satisfy HIPAA de-identification standards are not automatically sufficient for data to be considered anonymized or non-identifiable under Canadian laws. In many Canadian contexts, data that have undergone HIPAA-style de-identification may still be treated as coded or potentially identifiable. Consult your institution, privacy office, REB, or legal/privacy experts where appropriate.
<!-- #endregion -->

<!-- #region id="4v_KW8yItzXt" -->
# Install and Import Necessary Libraries

<!-- #endregion -->

<!-- #region id="n10Y0W8h4C1e" -->
Before running the notebook, some Python libraries may need to be installed and/or imported. *Installing* a library makes it available within your Python environment, while *importing* a library allows the notebook to actively use its functions within the code.

Some environments, such as Google Colab, already include commonly used libraries (e.g., NumPy and Matplotlib) pre-installed by default. Because of this, installation commands for those libraries are shown as commented lines (#) for reference only and do not need to be run in Colab. However, in a standard local Python environment, these libraries may need to be installed manually.

In contrast, pydicom is not included by default in many Python environments, so it must be installed before use. The exclamation mark (!) indicates that the command should be run as a system installation command within the notebook environment.
<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/"} id="OYeaNg3U9XqC" outputId="76873c20-9853-496a-a23d-1d8cd66b0e69"
# Install libraries
!pip install pydicom #the ! will be removed when taken off colab
# pip install numpy --> commented until taken off of Colab
# pip install matplotlib --> commented until taken off of Colab
# pip install pillow --> commented until taken off of Colab
# pip install hashlib --> commented until taken off of Colab
# pip install glob --> commented until taken off of Colab
```

```python id="Ro_xHC3grCn6"
# Import libraries
import os
import pydicom
from pydicom.data import get_testdata_files
from pydicom.uid import generate_uid
from hashlib import sha256
import matplotlib.pyplot as plt
from PIL import Image
import numpy as np
import glob
import random
```

<!-- #region id="6kP89TFNS39Q" -->
# Load helper utilities to show outputs from functions A-F (expand to see functions)
<!-- #endregion -->

<!-- #region id="C1IE3JCw8hOL" -->
The code below creates helper functions used later in the notebook. These functions do not change or de-identify the DICOM file. They simply help display selected metadata fields before and after a de-identification step, so users can more easily confirm what changed.

For example, the helper functions allow DICOM tags to be entered using either their keyword name, such as `PatientName`, or their numeric tag, such as `(0010,0010)`.
<!-- #endregion -->

```python id="VqTzGVagSwEi"
# ---- Helper functions for displaying metadata before and after de-identification ----
# These functions do not modify the DICOM file.
# They are only used to make the tutorial outputs easier to read.

from pydicom.tag import Tag
from pydicom.datadict import tag_for_keyword, keyword_for_tag


def format_dicom_tag(tag_input):
    """
    Convert different DICOM tag formats into a standardized format that Python can read.

    DICOM metadata fields can be written in different ways, including:
    - A keyword name, such as "PatientName"
    - A hexadecimal tag number, such as "0010,0010"
    - A tuple format, such as (0x0010, 0x0010)

    Hexadecimal numbers are commonly used in DICOM metadata because each
    DICOM field is assigned a unique tag identifier.
    """

    # Try to interpret the input as a pydicom Tag
    try:
        return Tag(tag_input)
    except Exception:
        pass

    # If the tag is written as a tuple/list, such as (0x0010, 0x0010)
    if isinstance(tag_input, (tuple, list)) and len(tag_input) == 2:
        return Tag(int(tag_input[0]), int(tag_input[1]))

    # If the tag is written as text
    if isinstance(tag_input, str):

        # Try to interpret it as a DICOM keyword, such as "PatientName"
        tag_number = tag_for_keyword(tag_input)
        if tag_number is not None:
            return Tag(tag_number)

        # Try to interpret it as a hexadecimal tag number, such as "0010,0010"
        cleaned_tag = (
            tag_input
            .replace(",", "")
            .replace("(", "")
            .replace(")", "")
            .replace(" ", "")
            .lower()
        )

        if len(cleaned_tag) == 8 and all(c in "0123456789abcdef" for c in cleaned_tag):
            return Tag(int(cleaned_tag[:4], 16), int(cleaned_tag[4:], 16))

    raise ValueError(f"Unrecognized DICOM tag: {tag_input}")


def format_dicom_tag_label(tag):
    """
    Create a readable label for a DICOM tag.

    Example:
    (0010,0010) PatientName
    """

    tag = format_dicom_tag(tag)
    keyword = keyword_for_tag(tag) or ""
    return f"({tag.group:04X},{tag.element:04X}) {keyword}".strip()


def get_metadata_values(ds, tags):
    """
    Return the values for selected DICOM metadata fields.
    If a field is not present in the file, it will say '<not present>'.
    """

    values = {}

    for tag_input in tags:
        tag = format_dicom_tag(tag_input)
        label = format_dicom_tag_label(tag)

        if tag in ds:
            values[label] = str(ds[tag].value)
        else:
            values[label] = "<not present>"

    return values


def print_before_after(ds_before, ds_after, tags):
    """
    Print selected DICOM metadata values before and after a change.
    This helps users confirm whether a de-identification step worked as intended.
    """

    print("BEFORE:")
    for label, value in get_metadata_values(ds_before, tags).items():
        print(f"  {label}: {value}")

    print("\nAFTER:")
    for label, value in get_metadata_values(ds_after, tags).items():
        print(f"  {label}: {value}")
```

<!-- #region id="JlFcAUPTKU2r" -->
# 1. Configuration --> edit this if you want to try the functions on your own DICOM file or use the pydicom sample file.
<!-- #endregion -->

<!-- #region id="deJZ_c_cKiGq" -->
User can choose to use their own dicom image or use the pydicom sample image for processing.
<!-- #endregion -->

```python id="k-yyXngptUOk"
USE_SAMPLE = False  # Set to False if you want to use your own image
custom_path = "/content/MRI_slice_1.dcm" # <-- update this path after dragging & dropping your dicom file into the Google Colab Files section.

if USE_SAMPLE:
    from pydicom.data import get_testdata_files
    dicom_path = get_testdata_files("CT_small.dcm")[0]
else:
    dicom_path = custom_path

ds = pydicom.dcmread(dicom_path) #dicom file to play with in the demos
ds_baseline = pydicom.dcmread(dicom_path)  #untouched copy for demos
```

<!-- #region id="XQi9SxaMuJDu" -->
# 2. Example identifiers commonly considered during DICOM de-identification

<!-- #endregion -->

<!-- #region id="ml7eH5p2rlru" -->
One commonly cited framework is the HIPAA Safe Harbor method, which identifies 18 categories of identifiers that are typically removed from health data. For DICOM files, examples may include:

1. Names
2. Geographic subdivisions smaller than a state
3. All elements of dates (except year)
4. Telephone numbers
5. Fax numbers
6. Email addresses
7. Social Security numbers
8. Medical record numbers (MRN)
9. Health plan beneficiary numbers
10. Account numbers
11. Certificate/license numbers
12. Vehicle identifiers
13. Device identifiers
14. URLs
15. IP addresses
16. Biometric identifiers
17. Full-face photos or comparable images
18. Any unique identifying number, characteristic, or code

Some of these may correspond to DICOM metadata fields such as:
- (0010,0010) PatientName
- (0008,0081) InstitutionAddress
- (0008,0022) Acquisition Date
- (0040,1103) PersonsTelephoneNumbers
- (0010,0020) PatientID
- (0010,1000) OtherPatientIDs
- (0010,0030) PatientBirthDate
- (0008,0090) ReferringPhysicianName
- (0008,0080) InstitutionName
- (0008,1070) OperatorsName
- (0018,1000) DeviceSerialNumber
- ... and many more

This is not an exhaustive list. The metadata elements that should be reviewed or modified depend on the file content, project context, jurisdiction, and intended data use.

In this tutorial, the HIPAA Safe Harbor identifiers are used primarily as a simple and widely recognized educational framework to help demonstrate common types of potentially identifying information that may appear in DICOM data. However, Canadian privacy frameworks and provincial regulations may differ from U.S. approaches and do not necessarily define a fixed list of identifiers in the same way. Users should refer to their institutional policies, REB requirements, privacy offices, and applicable provincial/federal guidance when determining appropriate de-identification practices for their project.
<!-- #endregion -->

<!-- #region id="Rz2fb5J7uV_p" -->
# 3. Load Sample DICOM File
<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/", "height": 428} id="iRP8O4wuuIwR" outputId="f4a0e91a-afae-4b82-c672-570b63a10340"
img = ds_baseline.pixel_array

# Display the image
plt.imshow(img, cmap='gray')
plt.title("Example original dicom image")
plt.axis('off')
plt.show()
```

<!-- #region id="sSemIvBAub9X" -->
#4. Inspect metadata that may contain personal or identifying information
<!-- #endregion -->

<!-- #region id="mZbphsRQH0na" -->
A great resource to reference DICOM metadata tags can be found here: https:/www.dicomlibrary.com/dicom/dicom-tags/



Review your DICOM file carefully for metadata fields that may contain direct identifiers (e.g., patient name or DOB), indirect identifiers (e.g., patient age or postal code), dates, site information, device information, or other elements that could contribute to re-identification risk.
<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/"} id="k1wa3JZoueUV" outputId="be350c1e-bbea-45f2-b020-1825c6311740"
print("\n--- ALL METADATA BEFORE DE-IDENTIFICATION ---")
for elem in ds:
    if elem.VR != "OB" and elem.tag.group != 0x7FE0:
        print(f"{elem.tag} {elem.name}: {elem.value}")
```

<!-- #region id="1OGb8kpTHMcz" -->
Metadata tags containing dates may include fields such as:

*   Study date
*   Series date
*   Acquisition date
*   Content date
*   Patient birth date
*   ... and so on.

Some dates may be directly identifying on their own, while others may increase re-identification risk when combined with other information. The example below is not a complete list. DICOM files can contain many date-related attributes, so the original identified file should always be reviewed carefully.

Below are some example metadata tags containing dates.
-----

**Please note that this is NOT a conclusive list** - the DICOM standard is constantly evolving, and many metadata tags may contain dates. You should always carefully inspect your original (identified) image to determine all fields that contain dates, and then add them to the helper function to de-identify them.
<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/"} id="FQJRTGE-HIbL" outputId="135e42ae-3d0a-4ccd-d0ad-703f6f0fd013"
date_tags = {
    (0x0008, 0x0020): "Study Date",
    (0x0008, 0x0021): "Series Date",
    (0x0008, 0x0022): "Acquisition Date",
    (0x0008, 0x0023): "Content Date",
    (0x0008, 0x0012): "Instance Creation Date",
    (0x0018, 0x1012): "Date of Secondary Capture", # May not exist in all files
    (0x0040, 0x0244): "Performed Procedure Step Start Date",  # May not exist in all files
    (0x0010, 0x0030): "Patient Birth Date"
}


#Show the output of the above-listed date tags.
print('Output of the original dicom file "date_tags":')
for tag, label in date_tags.items():
    if tag in ds_baseline:
        print(f"{label}: {ds_baseline[tag].value}")
    else:
        print(f"{label}: (not found)")
```

<!-- #region id="H-wcReZpJQHw" -->
## Date handling tip

When modifying image acquisition/study dates, users should apply a consistent approach across related date fields rather than generalizing one date, clearing another, and shifting another.

For example, if study-related dates are shifted, they should generally be shifted together so that the temporal relationships between the study, series, acquisition, and image instances are preserved. This helps maintain internal consistency within the DICOM file and may reduce issues when opening the de-identified file in another viewer or research workflow.

Where dates are needed for longitudinal imaging, follow-up studies, or approved record linkage, avoid applying inconsistent transformations across related date fields.
<!-- #endregion -->

<!-- #region id="4Abe86t7TlxL" -->
#Mapping dictionary to define which tags you want to apply functions A-F to.
<!-- #endregion -->

<!-- #region id="oEjjeMWt-kwM" -->
### Choosing which de-identification action to apply

***The block of code underneath is intended to be edited and customized by the user based on their own dataset and project requirements.***

The configuration map below tells the notebook which action to apply to selected DICOM metadata fields. There is no single “best” action for every field. The appropriate choice depends on the type of information, whether the information is needed for analysis, the intended data use, and your institution’s policies.

In general:
- `clear` is useful when a field is not needed and can be removed.
- `mask` is useful when a field should be replaced with a generic placeholder, such as `REDACTED`.
- `generalize` is useful when partial information is needed, such as keeping only the year from a full date.
- `pseudonymize` is useful when identifiers need to be replaced with consistent coded values so records can still be linked within a project.
- `noise` is useful when dates or numeric values need to be shifted or perturbed while preserving some general structure.
- `pixel_masks` are used to cover burned-in identifiers that appear directly on the image pixels.

-----
***Important:*** If you need to link de-identified files back to the original records, this requires a secure key or **linkage file**. That key should be stored separately, protected according to institutional policy, and **never shared publicly.** If a linkage key is retained, the data may still be considered coded or potentially identifiable depending on the context and applicable policies.
<!-- #endregion -->

```python id="bgSTQYNLTyRZ"
DEID_MAP = {
    "clear": [
        # Best for fields that are not needed and can be removed entirely
        "OperatorsName",
        "InstitutionAddress",
    ],

    "mask": {
        # Best for fields that should be replaced with a generic placeholder
        "InstitutionName": "REDACTED HOSPITAL",
    },

    "generalize": {
        # Best for fields where less-specific information is still useful
        # Example: keep only the year from a full date
        "PatientBirthDate": "keep_year",
    },

    "pseudonymize": [
        # Best for identifiers that need to be replaced with consistent coded values
        # This supports linking records within a project without showing the original identifier
        "PatientID",
    ],

    "noise": {
        # Best for shifting or perturbing dates/numeric values
        # Example: shift StudyDate randomly within ±2 years
        "StudyDate": {"type": "date_shift", "range": 2},
    },

    "suppress_private": True,
    # Removes private DICOM elements when applying the plan

    "pixel_masks": [
        # Optional blackout boxes for burned-in identifiers in the image pixels
        # Format: [x1, y1, x2, y2]
        # Example: [5, 5, 200, 40],
    ],
}
```

<!-- #region id="fNzTj5l8umqG" -->
# 5. Common de-identification techniques demonstrated in this notebook
<!-- #endregion -->

<!-- #region id="DUNrsdNKczAZ" -->
This section demonstrates several technical strategies that can be applied to DICOM metadata. These examples are intended to illustrate common approaches used in research workflows. Whether a given approach is appropriate or sufficient depends on the data, project context, intended data use, applicable governance requirements, and institutional policies. Users should consult their institution, REB, privacy office, and relevant provincial/federal guidance where appropriate.

### **Descriptions & Examples of Functions A-F are given below.**
<!-- #endregion -->

<!-- #region id="My1OgugMuniu" -->
# A. Direct Clearing of Tags

<!-- #endregion -->

<!-- #region id="gcpSxfWBWuaO" -->
Direct clearing means removing selected metadata elements entirely. This can be useful when a field is known to contain a direct identifier or other information that is not needed for the intended research use.

See below the python function followed by the demo.
<!-- #endregion -->

```python id="rkshnU8kulQJ"
def clear_tags(ds, tags_to_clear):
    for tag in tags_to_clear:
        T = format_dicom_tag(tag)
        if T in ds:
            del ds[T]
    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="R9G7yvedU--7" outputId="ccb886d9-90b8-47ee-ae35-8a6375125653"
_demoA_before = ds_baseline.copy()
_demoA_after = ds.copy()

_demoA_after = clear_tags(_demoA_after, ["InstitutionAddress"])

print("A) CLEAR example: remove InstitutionAddress and OperatorsName")
print_before_after(_demoA_before, _demoA_after, ["InstitutionAddress"])

```

<!-- #region id="MblXGg5BuuGO" -->
# B. Masking / Replacing Tags with Generic Values
<!-- #endregion -->

<!-- #region id="FSBCBii2hGbd" -->
Masking replaces an original value with a generic placeholder such as “REDACTED” or “UNKNOWN.” The field remains present, but the original value is no longer shown.

See the python function below, followed by a demo.
<!-- #endregion -->

```python id="R6NcL_-huu-X"
def mask_tags(ds, replacements):
    for tag, value in replacements.items():
        T = format_dicom_tag(tag)

        if T in ds:
            ds[T].value = value

        else:
            # Optional: create the field if it does not already exist
            try:
                kw = keyword_for_tag(T)

                if kw:
                    setattr(ds, kw, value)

            except Exception:
                pass

    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="Q5LeUVYbXJZl" outputId="6d89c972-2b41-467b-a716-e5d7309192e3"
_demoB_before = ds_baseline.copy()
_demoB_after = ds.copy()

_demoB_after = mask_tags(
    _demoB_after,
    {"InstitutionName": "REDACTED HOSPITAL"}
)

print("B) MASK example: InstitutionName → 'REDACTED HOSPITAL'")
print_before_after(
    _demoB_before,
    _demoB_after,
    ["InstitutionName"]
)
```

<!-- #region id="gtp0H09ruyp-" -->
# C. Generalization Example: Keeping only birth year
<!-- #endregion -->

<!-- #region id="FIwd3LMAhWRk" -->
Generalization reduces specificity while retaining some analytic usefulness.

For general applications, generalization may involve "binning" data into broader ranges to reduce specificity - e.g., binning the age 43 into the range 40 - 45. However, in the DICOM Standard, we are unable to store data in ranges because attributes such as Date (DA) must follow a strict YYYYMMDD format.

A practical way to demonstrate generalization is to preserve only the birth year and replace the month and day with a dummy value. This reduces specificity as the exact birth date is hidden, while still retaining useful coarse information (the year of birth).

See the python function below, followed by a demo.
<!-- #endregion -->

```python id="UjTeGP1TXUGh"
def generalize_date(ds, tag="PatientBirthDate", seed=None):
    """
    Keep the original year while replacing the month and day
    with randomly generated placeholder values.
    """

    if tag in ds and ds[tag].value:
        s = str(ds[tag].value)

        if len(s) >= 4 and s[:4].isdigit():
            r = random.Random(seed)

            mm = r.randint(1, 12)
            dd = r.randint(1, 27)

            ds[tag].value = f"{s[:4]}{mm:02d}{dd:02d}"

    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="48dDjD-LXWeR" outputId="a125bf9a-20f9-417b-cfd4-b0edf423c309"
_demoC_before = ds_baseline.copy()
_demoC_after = ds.copy()

_demoC_after = generalize_date(_demoC_after)

print("C) GENERALIZE example: keep birth year while replacing month/day")
print_before_after(
    _demoC_before,
    _demoC_after,
    ["PatientBirthDate"]
)
```

<!-- #region id="mDagzWJivnCO" -->
# D. Date shifting / perturbation
<!-- #endregion -->

<!-- #region id="b2Zp9u8Ghj0s" -->
Perturbation means modifying values slightly so that exact original values are hidden while some analytic utility is preserved. In DICOM workflows, a common example is shifting dates by a random offset. This can help protect identifiable calendar dates while preserving relative timing within a study.

See below for a demo.
<!-- #endregion -->

```python id="tWluXLXFawn5"
from datetime import datetime, timedelta
import random

def shift_date(ds, tag="StudyDate", max_shift_days=90):
    """
    Randomly shift a DICOM date forward or backward by a selected number of days.

    The output remains in valid YYYYMMDD DICOM date format.
    """

    if tag in ds and ds[tag].value:

        try:
            original = str(ds[tag].value)

            dt = datetime.strptime(original, "%Y%m%d")

            offset = random.randint(-max_shift_days, max_shift_days)

            new_dt = dt + timedelta(days=offset)

            ds[tag].value = new_dt.strftime("%Y%m%d")

        except Exception:
            pass  # Skip invalid or non-standard date formats

    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="AP4Ulnl7a0pQ" outputId="117e5d2b-b777-4206-f7a8-e56bb2aca8a7"
_demoD_before = ds_baseline.copy()
_demoD_after = ds.copy()

shift_date(_demoD_after, tag="StudyDate", max_shift_days=90)

print("D) DATE SHIFT example: randomly shift StudyDate by up to ±90 days")

print_before_after(
    _demoD_before,
    _demoD_after,
    ["StudyDate"]
)

```

<!-- #region id="EzGcaZ4Ivsc_" -->
# E. Pseudonymization: coded patient ID
<!-- #endregion -->

<!-- #region id="Zs1SewSgkRup" -->
Pseudonymization replaces an identifier with a consistent coded value, such as a hash. The same input produces the same pseudonym, which can support linkage across records without revealing the original value directly.

Pseudonymized data may still be considered identifiable, depending on whether re-linkage is possible and how the keying or coding process is governed.

See below for a demo.
<!-- #endregion -->

```python id="f0upFNKabCYY"
from hashlib import sha256


def generate_pseudonym(value):
    """
    Convert an identifier into a pseudonymized code using SHA-256 hashing.

    The original value is replaced with a shortened hash value
    that can help reduce direct identifiability.
    """

    return sha256(str(value).encode()).hexdigest()[:12]
```

```python colab={"base_uri": "https://localhost:8080/"} id="xjqpjGpQbD-v" outputId="e22e227f-c84c-4138-fbf7-98923ffae86e"
_demoE_before = ds_baseline.copy()
_demoE_after = ds.copy()

if "PatientID" in _demoE_after:
    _demoE_after.PatientID = generate_pseudonym(
        _demoE_after.PatientID
    )

print("E) PSEUDONYMIZE example: PatientID → pseudonymized hash")

print_before_after(
    _demoE_before,
    _demoE_after,
    ["PatientID"]
)
```

<!-- #region id="kdYiBtYiv9Yb" -->
# F. Suppression: removing fields that may increase identification risk
<!-- #endregion -->

<!-- #region id="5Pj967Z-ksrb" -->
Suppression is a general de-identification technique in which data elements are removed so they cannot be used directly or indirectly to identify an individual. In practice, suppression may be applied in different ways, such as:


*   Broad removal of categories of fields that may contain identifying content, such as private DICOM elements

*   Selective removal of fields whose presence may increase re-identification risk in a particular context



In this notebook, suppression is demonstrated by removing private DICOM elements. This is presented as a general technical strategy used in de-identification workflows. It is not intended to suggest that “suppression” is a formally defined HIPAA term. Rather, HIPAA interpretive guidance discusses the removal of identifiers and the need to reduce the risk that individuals can be identified from the remaining data.

We separate suppression here from direct clearing by scope:

*   direct clearing removes specific known fields selected by the user

*   suppression removes broader groups of fields or fields that may become identifying in context

----


See below for a demo.
<!-- #endregion -->

```python id="9BLsIXyybLAz"
def suppress_private(ds):
    """
    Remove private DICOM metadata elements.

    In DICOM files, private/vendor-specific fields are commonly stored
    in odd-numbered groups. These fields may contain additional
    identifying or institution-specific information.
    """

    tags_to_remove = []

    for tag in list(ds.keys()):

        # Private DICOM elements are typically stored in odd-numbered groups
        if tag.group % 2 != 0:
            tags_to_remove.append(tag)

    for tag in tags_to_remove:
        del ds[tag]

    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="Tqdd_UFYbNFw" outputId="bb286836-12c7-46d7-c713-d1013aa8f716"
_demoF = ds.copy()

# Count private elements before suppression
n_before = sum(1 for e in _demoF if e.tag.group % 2 == 1)

# Remove private elements
_demoF = suppress_private(_demoF)

# Count remaining private elements
n_after = sum(1 for e in _demoF if e.tag.group % 2 == 1)

print(f"F) SUPPRESS example: private DICOM elements {n_before} → {n_after}")

```

<!-- #region id="AI29BwTx4nkE" -->
# 8. Handling Burned-in Annotations in Pixel Data

Some DICOM files contain identifying information directly in the image pixels rather than only in the metadata. Examples may include patient names, accession numbers, dates, or site identifiers burned into the image.

Two possible strategies:
1. Masking using black boxes
2. Cropping the image

Pixel-based identifiers should be reviewed carefully, especially in modalities such as ultrasound or secondary capture images. Please note in volumetric and/or video data, ***every*** slice needs to be carefully checked for residual burned-in text identifiers.
<!-- #endregion -->

```python id="Zp-kEC29k_nn"
# Path to the ultrasound DICOM file
# Update this path after uploading your file into the Google Colab Files section

US_path = "/content/ultrasound_image1.dcm"

# Load the ultrasound DICOM file
us_ds = pydicom.dcmread(US_path)

# Create an untouched copy for before/after demonstrations
us_ds_baseline = pydicom.dcmread(US_path)
```

<!-- #region id="S_hVPdFKlZ_s" -->
## Original Ultrasound Image
<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/", "height": 338} id="u-LBahwIlWUc" outputId="da6fc1a8-1a06-42a6-899b-54352b160abf"
# ---- Display original ultrasound image ----

us_img = us_ds_baseline.pixel_array

plt.imshow(us_img, cmap="gray")

plt.title("Example ultrasound image (original)")

plt.axis("off")
plt.show()
```

```python id="Q1NBvBAOgHlf"
# ---- Pixel-based de-identification examples ----
# These examples demonstrate simple approaches for removing
# burned-in identifiers from image pixels.

def crop_image(ds, start_row):
    """
    Crop the image by removing rows from the top portion of the image.

    This can be useful when identifiers appear in a banner/header region.
    """

    if "PixelData" not in ds:
        return None

    arr = ds.pixel_array

    # Keep image rows from start_row to the bottom
    cropped = arr[start_row:, :]

    return cropped


def blackout_region(ds, x1, y1, x2, y2):
    """
    Apply a black box over a selected image region.

    This can be useful for covering burned-in identifiers
    directly within the image pixels.
    """

    if "PixelData" not in ds:
        return None

    arr = ds.pixel_array.copy()

    # Replace selected region with black pixels
    arr[y1:y2, x1:x2] = 0

    return arr
```

```python colab={"base_uri": "https://localhost:8080/", "height": 319} id="23cuN-GmgKwT" outputId="4af9cedd-62c2-4f6f-c107-9bbf6ce3829e"
# ---- Example: crop the top portion of the image ----

if "PixelData" in us_ds:

    # USER INPUT:
    # Adjust this value depending on where burned-in text appears
    CROP_START_ROW = 71

    cropped_image = crop_image(us_ds, CROP_START_ROW)

    # Display cropped result
    plt.imshow(cropped_image, cmap="gray")

    plt.title(
        f"Cropped image (removed top {CROP_START_ROW} rows)"
    )

    plt.axis("off")
    plt.show()
```

<!-- #region id="qgwsr7uuLUIq" -->
## Multi-frame data (multi-slice or video)
<!-- #endregion -->

<!-- #region id="roNowMlJLbC5" -->
Many DICOM datasets contain more than one image per acquisition. For example, CT and MRI studies are often volumetric acquisitions containing multiple image slices across a larger anatomical volume, while ultrasound imaging is commonly stored as multi-frame/video data. Depending on the imaging modality, vendor, software version, and acquisition format, these images may be stored either as multiple individual DICOM files within a folder (e.g., one file per slice/frame) or as a single multi-frame DICOM file containing all slices/frames together.


In all cases, **all slices and frames should be carefully reviewed** for potentially identifying information, including burned-in text within the pixel data. **If cropping or black-box masking is applied, it should be applied to all relevant slices/frames in the acquisition rather than only a single image**.

Users should always inspect the output carefully before saving, sharing, or publishing de-identified files and should refer to their institutional policies and governance requirements where appropriate.
<!-- #endregion -->

```python id="23CpkLQvNIE9"
# Path to the multi-frame ultrasound DICOM file
# Update this path after uploading your file into the Google Colab Files section

US_path = "/content/ultrasound_multiframe_color.dcm"

# Load the ultrasound DICOM file
us_ds = pydicom.dcmread(US_path)

# Create an untouched copy for before/after demonstrations
us_ds_baseline = pydicom.dcmread(US_path)
```

```python id="MEILJ6NVM4TS"
### Pixel de-identification for multi-frame/video DICOM data

def crop_all_frames(ds, start_row):
    """
    Crop the top portion of every frame in a DICOM image.

    This is useful for multi-frame/video DICOM files where burned-in
    identifiers appear in the same location across all frames.
    """

    if "PixelData" not in ds:
        return None

    arr = ds.pixel_array

    # Single-frame image: rows x columns
    if arr.ndim == 2:
        cropped = arr[start_row:, :]

    # Multi-frame grayscale image: frames x rows x columns
    elif arr.ndim == 3:
        cropped = arr[:, start_row:, :]

    # Multi-frame colour image: frames x rows x columns x colour channels
    elif arr.ndim == 4:
        cropped = arr[:, start_row:, :, :]

    else:
        raise ValueError("Unexpected pixel array shape. Please inspect ds.pixel_array.shape.")

    return cropped
```

```python id="12b_nXyvNnyJ"
def apply_cropped_pixels_to_dicom(ds, cropped_pixels):
    """
    Replace the DICOM pixel data with cropped pixel data.

    This updates the PixelData, Rows, and Columns fields so the file
    remains consistent with the new cropped image size.
    """

    ds.PixelData = cropped_pixels.tobytes()

    # Rows and columns are always the last image height/width dimensions
    if cropped_pixels.ndim == 2:
        ds.Rows, ds.Columns = cropped_pixels.shape

    elif cropped_pixels.ndim == 3:
        ds.Rows, ds.Columns = cropped_pixels.shape[1], cropped_pixels.shape[2]

    elif cropped_pixels.ndim == 4:
        ds.Rows, ds.Columns = cropped_pixels.shape[1], cropped_pixels.shape[2]

    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="uaMBOORPNDO2" outputId="daefea8e-6fd3-4436-f1c8-15ac067641a3"
# Apply cropped pixel data to a copy of the DICOM file
us_ds_cropped = us_ds.copy()

cropped_frames = crop_all_frames(us_ds_cropped, CROP_START_ROW)

us_ds_cropped = apply_cropped_pixels_to_dicom(
    us_ds_cropped,
    cropped_frames
)

# Optional: save cropped DICOM
output_path = "cropped_ultrasound_output.dcm"

us_ds_cropped.save_as(output_path)

print(f"Saved cropped DICOM to: {output_path}")
```

<!-- #region id="oka5SAaS5OLW" -->
# 7. Process and Save Your De-identified File
<!-- #endregion -->

<!-- #region id="kXQ4IeyXbf7P" -->
###Apply the mapping table to de-id selected metadata tags.
<!-- #endregion -->

<!-- #region id="usdMlarcemrl" -->
Note:
Applying these transformations does not on its own guarantee that the output file is non-identifiable under any specific legal or institutional standard. Always review both metadata and pixel data, assess residual re-identification risk, and validate the output before sharing or publishing.
<!-- #endregion -->

```python id="IjkKtUqcbeqv"
# === Apply the DEID_MAP plan ===
# This function applies the actions selected in the DEID_MAP configuration section above.

def apply_plan(ds, plan):

    # A) Clear selected fields
    if plan.get("clear"):
        ds = clear_tags(ds, plan["clear"])

    # B) Mask selected fields
    if plan.get("mask"):
        ds = mask_tags(ds, plan["mask"])

    # C) Generalize selected date fields
    if plan.get("generalize"):
        for tag, action in plan["generalize"].items():
            if action == "keep_year":
                ds = generalize_date(ds, tag=tag)

    # D) Shift or perturb selected date fields
    if plan.get("noise"):
        for tag, settings in plan["noise"].items():
            if settings.get("type") == "date_shift":
                max_shift = settings.get("range", 90)
                ds = shift_date(ds, tag=tag, max_shift_days=max_shift)

    # E) Pseudonymize selected fields
    if plan.get("pseudonymize"):
        for tag in plan["pseudonymize"]:
            T = format_dicom_tag(tag)

            if T in ds:
                ds[T].value = generate_pseudonym(ds[T].value)

    # F) Suppress private DICOM elements
    if plan.get("suppress_private"):
        ds = suppress_private(ds)

    # G) Apply pixel masks for burned-in identifiers
    if plan.get("pixel_masks") and "PixelData" in ds:
        arr = ds.pixel_array.copy()

        for x1, y1, x2, y2 in plan["pixel_masks"]:
            arr[y1:y2, x1:x2] = 0

        ds.PixelData = arr.tobytes()
        ds.Rows, ds.Columns = arr.shape[:2]

    return ds

```

```python id="FaqYsn094i_W" colab={"base_uri": "https://localhost:8080/"} outputId="a6242559-d995-4ad9-993b-a3115a9336fb"
# Apply the de-identification plan to a copy of the DICOM file
ds_plan = ds.copy()
ds_plan = apply_plan(ds_plan, DEID_MAP)

# Preview selected fields after applying the plan
print("AFTER applying DEID_MAP (sample of fields):")

for label, value in get_metadata_values(
    ds_plan,
    ["PatientID", "PatientName", "PatientBirthDate", "StudyDate", "InstitutionName"]
).items():
    print(f"  {label}: {value}")
```

<!-- #region id="0FBt2pZpHz_l" -->
Save your file
<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/"} id="W4Pxv_4HHzsJ" outputId="cc00db4d-66fd-480a-ef34-be25fa125d42"
# Save the de-identified DICOM file
output_path = "deidentified_output.dcm" #user to adjust pathing

ds_plan.save_as(output_path)

print(f"\nSaved de-identified DICOM to: {output_path}")
```

<!-- #region id="JpMU2y5I5KJM" -->
# 9. Final Notes and Resources
<!-- #endregion -->

<!-- #region id="OccXAS1c56I-" -->
Other useful tools/libraries:
- `dicom-anonymizer` (https://pypi.org/project/dicom-anonymizer/)
- `pyminc`, `nibabel` for NIfTI conversions
- `dcm2niix` (command-line tool for research pipelines)
- `pynetdicom` for secure DICOM transfer

-----
Always validate and review output files before saving, sharing, or publishing them. De-identification should include review of both metadata and pixel data. Users should:


*   Verify that intended metadata fields/tags were properly modified or removed

*   Review all remaining metadata fields for residual identifying information


*   Preview images to check for burned-in text or identifiers within the pixel data

*   Confirm that the transformations applied are appropriate for the project context, intended data use, governance requirements, institutional policies, and jurisdiction







For projects involving Canadian data, remember that technical de-identification steps alone may not be enough for data to be considered anonymized or non-identifiable under law or institutional policy.
<!-- #endregion -->

<!-- #region id="BL5P_FeQQ_ps" -->
# References & Acknowledgements
<!-- #endregion -->

<!-- #region id="MoIGYoL0RD2h" -->
This tutorial makes use of several open-source Python libraries and tools, including:

**Python libraries/packages:**
*   pydicom — https://pydicom.github.io/
*   NumPy — https://numpy.org/
*   Matplotlib — https://matplotlib.org/
*   Pillow (PIL) — https://python-pillow.org/
*   hashlib (Python Standard Library) — https://docs.python.org/3/library/hashlib.html


**Additional DICOM/de-identification resources referenced:**
*   DICOM Standard — https://www.dicomstandard.org/
*   Information and Privacy Commissioner of Ontario (IPC) De-Identification Guidelines - https://www.ipc.on.ca/en/resources/de-identification-guidelines-structured-data
*   Personal Information Protection and Electronic Documents Act (PIPEDA) - https://www.priv.gc.ca/en/privacy-topics/privacy-laws-in-canada/the-personal-information-protection-and-electronic-documents-act-pipeda/
*   HIPAA Safe Harbor Guidance — https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/index.html


----

Acknowledgement:
This notebook was developed as part of educational and de-identification resource initiatives associated with the ARCHIMEDES platform.
<!-- #endregion -->
