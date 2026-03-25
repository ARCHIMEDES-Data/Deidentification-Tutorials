---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.18.1
  kernelspec:
    display_name: Python 3
    name: python3
---

<!-- #region id="Erl8u-UXtf3N" -->
# De-Identification of DICOM Data - Jupyter Notebook Tutorial

This script-style Jupyter notebook introduces researchers to Python-based tools for reducing identifiability in DICOM data. It demonstrates common technical approaches used in de-identification workflows, with examples informed in part by HIPAA Safe Harbor concepts for identifiers commonly removed from health data.

It allows users to:
- Use either a sample or custom DICOM file
- Choose specific tags to clear, mask, generalize, pseudonymize, or suppress
- Filter and batch process files from a folder

----
# NOTE:
This tutorial is provided for educational and research workflow support only.
**It does not provide legal advice, and it does not by itself determine whether data are de-identified, anonymized, or otherwise compliant under any specific law or institutional policy.**
Users are responsible for ensuring that their data handling practices meet the requirements of the laws, regulations, ethics approvals, data sharing agreements, and institutional policies applicable to their project.

**For Canadian users:**
Methods commonly used to satisfy HIPAA de-identification standards are not automatically sufficient for data to be considered anonymized or non-identifiable under Canadian laws. In many Canadian contexts, data that have undergone HIPAA-style de-identification may still be treated as coded or potentially identifiable. Consult your institution, privacy office, REB, or legal/privacy experts where appropriate.
<!-- #endregion -->

<!-- #region id="4v_KW8yItzXt" -->
# Install and Import Necessary Libraries

<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/"} id="OYeaNg3U9XqC" outputId="4e7ad0f1-751d-4801-a9e9-2b36374d9342"
!pip install pydicom #the ! will be removed when taken off colab
# pip install numpy --> commented until taken off of Colab
# pip install matplotlib --> commented until taken off of Colab
# pip install pillow --> commented until taken off of Colab
# pip install hashlib --> commented until taken off of Colab
# pip install glob --> commented until taken off of Colab
```

```python id="Ro_xHC3grCn6"
# @title
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
#Load helper utilities to show outputs from functions A-F (expand to see functions)
<!-- #endregion -->

```python id="VqTzGVagSwEi"
# ---- Helper utilities for friendly demos (non-coder-friendly) ----
from pydicom.tag import Tag
from pydicom.datadict import tag_for_keyword, keyword_for_tag

def _resolve_tag(tag_like):
    """Accept 'PatientName', (0x0010,0x0010), [0x0010,0x0010], or pydicom Tag."""
    if isinstance(tag_like, Tag):
        return tag_like
    if isinstance(tag_like, (tuple, list)) and len(tag_like) == 2:
        return Tag(tag_like[0], tag_like[1])
    if isinstance(tag_like, str):
        # try keyword
        t = tag_for_keyword(tag_like)
        if t is not None:
            return Tag(t)
        # try hex string forms like "00100010" or "0010,0010"
        s = tag_like.replace(",", "").replace("(", "").replace(")", "").replace(" ", "").lower()
        if len(s) == 8 and all(c in "0123456789abcdef" for c in s):
            return Tag(int(s[:4], 16), int(s[4:], 16))
    raise ValueError(f"Unrecognized tag: {tag_like}")

def _label(tag):
    try:
        return f"({tag.group:04X},{tag.element:04X}) {keyword_for_tag(tag) or ''}".strip()
    except Exception:
        return str(tag)

def values_of(ds, tags):
    out = {}
    for t in tags:
        T = _resolve_tag(t)
        out[_label(T)] = str(ds[T].value) if T in ds else "<not present>"
    return out

def print_before_after(ds_before, ds_after, tags):
    print("BEFORE:")
    for k, v in values_of(ds_before, tags).items():
        print(f"  {k}: {v}")
    print("\nAFTER:")
    for k, v in values_of(ds_after, tags).items():
        print(f"  {k}: {v}")


from pydicom.tag import Tag
from pydicom.datadict import tag_for_keyword, keyword_for_tag

def _resolve_tag(tag_like):
    """
    Robustly convert many inputs → pydicom Tag:
      - 'PatientName' (keyword)
      - (0x0010,0x0010) or [0x0010,0x0010]
      - '00100010' or '0010,0010'
      - Tag / BaseTag / int-like accepted by Tag()
    """
    # 1) If Tag() can already handle it, use that path
    try:
        return Tag(tag_like)
    except Exception:
        pass

    # 2) tuple/list of two ints
    if isinstance(tag_like, (tuple, list)) and len(tag_like) == 2:
        return Tag(int(tag_like[0]), int(tag_like[1]))

    # 3) keyword or hex string
    if isinstance(tag_like, str):
        # try keyword
        t = tag_for_keyword(tag_like)
        if t is not None:
            return Tag(t)
        # try hex strings like "00100010" or "0010,0010"
        s = tag_like.replace(",", "").replace("(", "").replace(")", "").replace(" ", "").lower()
        if len(s) == 8 and all(c in "0123456789abcdef" for c in s):
            return Tag(int(s[:4], 16), int(s[4:], 16))

    raise ValueError(f"Unrecognized tag: {tag_like}")

```

<!-- #region id="JlFcAUPTKU2r" -->
# 1. Configuration --> edit this if you want to try the functions on your own DICOM file or use the pydicom sample file.
<!-- #endregion -->

<!-- #region id="deJZ_c_cKiGq" -->
User can choose to use their own dicom image or use the pydicom sample image for processing.
<!-- #endregion -->

```python id="k-yyXngptUOk"
USE_SAMPLE = False  # Set to False if you want to use your own image
custom_path = "/content/TEST_04Nov2025.MR.THORAX_Cardiac.1001.3.2025.11.04.12.54.56.434.53911847.dcm" # <-- update this path after dragging & dropping your dicom file into the Google Colab Files section.

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

This is not a complete or universally sufficient list. The metadata elements that should be reviewed or modified depend on the file content, project context, jurisdiction, and intended data use.
<!-- #endregion -->

<!-- #region id="Rz2fb5J7uV_p" -->
# 3. Load Sample DICOM File
<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/", "height": 428} id="iRP8O4wuuIwR" outputId="3112c39a-b162-42a2-abee-f3cb30b3cdd7"
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



Review your DICOM file carefully for metadata fields that may contain direct identifiers, indirect identifiers, dates, site information, device information, or other elements that could contribute to re-identification risk.
<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/"} id="k1wa3JZoueUV" outputId="50b0682d-4ff2-4862-8dcb-457af9343a3c"
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

**Please note that this is NOT a conclusive list** - the DICOM standard is constantly evolving, and many metadata tags may contain dates. You should always carefully inspect your original (identified) image to determine all fields that contain dates.
<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/"} id="FQJRTGE-HIbL" outputId="1b66d8af-3229-4d66-ab25-0e42e573dbc0"
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

<!-- #region id="4Abe86t7TlxL" -->
#Mapping dictionary to define which tags you want to apply functions A-F to.
<!-- #endregion -->

```python id="bgSTQYNLTyRZ"
# === Big configuration map: which tags to clear vs mask vs generalize vs pseudonymize vs noise ===
# People can edit this dictionary only, and then "apply plan" near the end.
DEID_MAP = {
    "clear": [
        # Examples: remove these fields entirely
        # "InstanceCreationDate",
        # "OperatorsName",
    ],
    "mask": {
        # Overwrite with a literal placeholder
        # "InstitutionName": "REDACTED HOSPITAL",
    },
    "generalize": {
        # Reduce specificity, e.g., keep year only
        # "PatientBirthDate": "keep_year",
    },
    "pseudonymize": [
        # Hash these identifiers (stable, non-reversible)
        # "PatientID",
    ],
    "noise": {
        # Add random perturbation. Example: ±2 years on StudyDate
        # "StudyDate": {"type": "date_shift", "range": 2}
    },
    "suppress_private": True,   # remove private/odd-group elements when applying plan
    "pixel_masks": [
        # Optional blackout boxes [x1, y1, x2, y2] for burned-in PHI in pixels
        # [5, 5, 200, 40],
    ],
    "salt": "ARCHIMEDES_DEMO_SALT"  # used for pseudonym hashing (optional)
}
```

<!-- #region id="fNzTj5l8umqG" -->
# 5. Common de-identification techniques demonstrated in this notebook
<!-- #endregion -->

<!-- #region id="DUNrsdNKczAZ" -->
This section demonstrates several technical strategies that can be applied to DICOM metadata. These examples are intended to illustrate common approaches used in research workflows. Whether a given approach is appropriate or sufficient depends on the data, context, and applicable governance requirements.
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
        T = _resolve_tag(tag)
        if T in ds:
            del ds[T]
    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="R9G7yvedU--7" outputId="9d1e74db-cad1-4f8d-9739-834a3faa6278"
_demoA_before = ds_baseline.copy()
_demoA_after = ds.copy()
_demoA_after = clear_tags(_demoA_after, ["InstanceCreationDate"])
print("A) CLEAR example: remove InstanceCreationDate")
print_before_after(_demoA_before, _demoA_after, ["InstanceCreationDate"])

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
        T = _resolve_tag(tag)
        if T in ds:
            ds[T].value = value
        else:
            # try to create if missing (optional)
            try:
                kw = keyword_for_tag(T)
                if kw:
                    setattr(ds, kw, value)
            except Exception:
                pass
    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="Q5LeUVYbXJZl" outputId="c6789a51-ea90-4c3e-c2c1-de51a61f0c97"
_demoB_before = ds_baseline.copy()
_demoB_after = ds.copy()
_demoB_after = mask_tags(_demoB_after, {"InstitutionName": "REDACTED HOSPITAL"})
print("B) MASK example: InstitutionName → 'REDACTED HOSPITAL'")
print_before_after(_demoB_before, _demoB_after, ["InstitutionName"])
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
    """Keep YYYY; replace MMDD with a valid fake (MM=01–12, DD=01–27)."""
    if tag in ds and ds[tag].value:
        s = str(ds[tag].value)
        if len(s) >= 4 and s[:4].isdigit():
            r = random.Random(seed)
            mm = r.randint(1, 12)
            dd = r.randint(1, 27)
            ds[tag].value = f"{s[:4]}{mm:02d}{dd:02d}"
    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="48dDjD-LXWeR" outputId="4557ad27-6f69-42d7-d3c0-0b4bb71fe971"
_demoC_before = ds_baseline.copy()
_demoC_after  = ds.copy()
_demoC_after  = generalize_date(_demoC_after)
print("C) GENERALIZE example: PatientBirthDate → YYYY0701")
print_before_after(_demoC_before, _demoC_after, ["PatientBirthDate"])
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
    Shift a DICOM date (DA) by ±max_shift_days.
    Keeps valid YYYYMMDD format.
    """
    if tag in ds and ds[tag].value:
        try:
            original = str(ds[tag].value)
            dt = datetime.strptime(original, "%Y%m%d")
            offset = random.randint(-max_shift_days, max_shift_days)
            new_dt = dt + timedelta(days=offset)
            ds[tag].value = new_dt.strftime("%Y%m%d")
        except Exception:
            pass  # if not a valid DA, skip
    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="AP4Ulnl7a0pQ" outputId="89784b80-6411-46a3-fc03-58d08973cffd"
_demoD_before = ds_baseline.copy()
_demoD_after  = ds.copy()

shift_date(_demoD_after, tag="StudyDate", max_shift_days=90)

print("D) DATE SHIFT example: StudyDate shifted randomly ±90 days")
print_before_after(_demoD_before, _demoD_after, ["StudyDate"])

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
def generate_pseudonym(value, salt=""):
    return sha256((salt + str(value)).encode()).hexdigest()[:12]  # 12 chars

# Your original example kept:
if 'PatientID' in ds:
    pass  # we'll show it in the demo below
```

```python colab={"base_uri": "https://localhost:8080/"} id="xjqpjGpQbD-v" outputId="b3e99e26-592a-40e9-ae6f-d780da4a6aa3"
_demoE_before = ds_baseline.copy()
_demoE_after = ds.copy()
if 'PatientID' in _demoE_after:
    _demoE_after.PatientID = generate_pseudonym(_demoE_after.PatientID, salt=DEID_MAP.get("salt", ""))
print("E) PSEUDONYMIZE example: PatientID → hash (12 chars)")
print_before_after(_demoE_before, _demoE_after, ["PatientID"])

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
    # remove odd-group (private) elements
    to_del = []
    for tag in list(ds.keys()):
        if tag.group % 2 != 0:
            to_del.append(tag)
    for t in to_del:
        del ds[t]
    return ds
```

```python colab={"base_uri": "https://localhost:8080/"} id="Tqdd_UFYbNFw" outputId="1e6474a7-7c5a-4ee6-cbc2-bb3484774ae9"
_demoF = ds.copy()
n_before = sum(1 for e in _demoF if e.tag.group % 2 == 1)
_demoF = suppress_private(_demoF)
n_after = sum(1 for e in _demoF if e.tag.group % 2 == 1)
print(f"F) SUPPRESS example: private elements {n_before} → {n_after}")

```

<!-- #region id="AI29BwTx4nkE" -->
# 8. Handling Burned-in Annotations in Pixel Data

Some DICOM files contain identifying information directly in the image pixels rather than only in the metadata. Examples may include patient names, accession numbers, dates, or site identifiers burned into the image.

Two possible strategies:
1. Masking using black boxes
2. Cropping the image

Pixel-based identifiers should be reviewed carefully, especially in modalities such as ultrasound or secondary capture images.
<!-- #endregion -->

```python id="Zp-kEC29k_nn"
US_path = "/content/ultrasound_image1.dcm" # <-- update this path after dragging & dropping your dicom file into the Google Colab Files section.

us_ds = pydicom.dcmread(US_path) #dicom file to play with in the demos
us_ds_baseline = pydicom.dcmread(US_path)  #untouched copy for demos
```

<!-- #region id="S_hVPdFKlZ_s" -->
## Original Ultrasound Image
<!-- #endregion -->

```python colab={"base_uri": "https://localhost:8080/", "height": 338} id="u-LBahwIlWUc" outputId="779303be-bab5-475d-f60c-ac584a3d5181"
us_img = us_ds_baseline.pixel_array

# Display the image
plt.imshow(us_img, cmap='gray')
plt.title("Example Ultrasound image (original)")
plt.axis('off')
plt.show()
```

```python id="Q1NBvBAOgHlf"
import random

def crop_image(ds, x1, y1, x2, y2):
    """Return a cropped numpy array from PixelData (ds.pixel_array)."""
    if 'PixelData' not in ds:
        return None
    arr = ds.pixel_array
    return arr[y1:y2, x1:x2]

def blackout_region(ds, x1, y1, x2, y2):
    """Return a copy of the pixel array with a black box applied."""
    if 'PixelData' not in ds:
        return None
    arr = ds.pixel_array.copy()
    arr[y1:y2, x1:x2] = 0
    return arr

```

```python colab={"base_uri": "https://localhost:8080/", "height": 319} id="23cuN-GmgKwT" outputId="802b0c2d-ab2b-416c-e899-c4d2d2424aa8"
if 'PixelData' in ds:
    arr = us_ds.pixel_array

    # === USER INPUT ===
    CROP_START_ROW = 71  # change this value (e.g., where the top blue bar ends)

    # Crop from that row to the bottom
    cropped = arr[CROP_START_ROW:, :]

     # Display result
    plt.imshow(cropped, cmap='gray')
    plt.title(f"Cropped image (removed top {CROP_START_ROW} rows)")
    plt.axis('off')
    plt.show()

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

```python colab={"base_uri": "https://localhost:8080/"} id="IjkKtUqcbeqv" outputId="4bdcf49a-cf6b-4e01-bbd6-58654c69b7b6"
# === Apply the DEID_MAP plan (optional one-click step) ===
def apply_plan(ds, plan):
    # A) clear
    if plan.get("clear"):
        ds = clear_tags(ds, plan["clear"])
    # B) mask
    if plan.get("mask"):
        ds = mask_tags(ds, plan["mask"])
    # C) generalize
    if plan.get("generalize"):
        ds = generalize_date(ds, plan["generalize"])
   # D) date shift / perturbation
    if plan.get("noise"):
      for tag, settings in plan["noise"].items():
        if settings.get("type") == "date_shift":
            max_shift = settings.get("range", 90)
            ds = shift_date(ds, tag=tag, max_shift_days=max_shift)
    # E) pseudonymize
    if plan.get("pseudonymize"):
        for t in plan["pseudonymize"]:
            T = _resolve_tag(t)
            if T in ds:
                ds[T].value = generate_pseudonym(ds[T].value, salt=plan.get("salt", ""))
    # F) suppress private
    if plan.get("suppress_private"):
        ds = suppress_private(ds)
    # Pixel masks (burned-in)
    if plan.get("pixel_masks"):
        if 'PixelData' in ds:
            arr = ds.pixel_array.copy()
            for (x1, y1, x2, y2) in plan["pixel_masks"]:
                arr[y1:y2, x1:x2] = 0
            ds.PixelData = arr.tobytes()
            ds.Rows, ds.Columns = arr.shape[:2]
    return ds

# Run-once example: apply plan to a copy and preview a few fields
ds_plan = ds.copy()
ds_plan = apply_plan(ds_plan, DEID_MAP)
print("AFTER applying DEID_MAP (sample of fields):")
for k, v in values_of(ds_plan, ["PatientID", "PatientName", "PatientBirthDate", "StudyDate", "InstitutionName"]).items():
    print(f"  {k}: {v}")

```

```python id="FaqYsn094i_W" colab={"base_uri": "https://localhost:8080/"} outputId="219343ca-821d-4849-97bb-9d8caa5ba86d"
output_path = "deidentified_output.dcm"
ds.save_as(output_path)
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
Always validate your output files before sharing or publishing. Review metadata, inspect pixel data for burned-in identifiers, and confirm that the transformations applied are appropriate for your project, governance requirements, and jurisdiction.

For projects involving Canadian data, remember that technical de-identification steps alone may not be enough for data to be considered anonymized or non-identifiable in law or policy.

<!-- #endregion -->
