# **De-Identification of DICOM Data - Jupyter Notebook Tutorial**

**Authors:**
Nadia Farrag, Ph.D. &
Jefferson Casimir

---

This script-style Jupyter notebook introduces researchers to Python-based tools for reducing identifiability in DICOM data. It demonstrates common technical approaches used in de-identification workflows, with examples informed in part by the Information and Privacy Commissioner of Ontario (IPC) & HIPAA Safe Harbor concepts for identifiers commonly removed from health data. It uses existing python libraries such as ``pydicom``, ``matplotlib``, ``numpy``, etc. Please see the ``ipynb`` Jupyter notebook for references.

> *This tutorial assumes a basic working knowledge of Python coding and familiarity with running Jupyter notebooks.*

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

------------
# Run in Google Colab

This tutorial can be run directly in Google Colab without installing Python, MyST, or JupyterLab locally.

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/ARCHIMEDES-Data/Deidentification-Tutorials/blob/main/src/DeID_Tutorial_DICOM.ipynb?forceRefresh=true)

## Getting Started

1. Click the “Open in Colab” button above.
2. Select:

   ```text
   Runtime → Run all
   ```

3. Follow the tutorial cells step-by-step.

The notebook uses sample MRI and ultrasound image files included in this GitHub repository.

The ultrasound video section is optional. To run it, users must upload their own de-identified ultrasound video DICOM file when prompted.

> Important: Do not upload identifiable patient data unless you have appropriate approvals and institutional guidance.

------------
# Install Guide (MyST)

In the terminal, navigate to the root directory of this project, create a virtual environment, activate it, and install the dependencies:

```bash
python -m venv venv
source venv/bin/activate
pip install -e .
```

## Run Jupyter Server (Optional)

This enables launching notebooks directly from MyST.

```bash
jupyter lab --NotebookApp.token=0123456789abcdef0123456789abcdef0123456789abcdef --NotebookApp.allow_origin='http://localhost:3000'
```

Notes:

- The token value can be changed, but it must match the `token` field in `myst.yml`.
- The `allow_origin` uses the default MyST port (`3000`). This will work as long as another service is not already bound to that port.
- If another service is using port `3000`, modify the command above accordingly.
- Jupyter Server will print the port it is running on. If it is not `8888`, update the `url` field in `myst.yml`.

This terminal will now continue running JupyterLab. Open a new terminal window for running MyST.

## Run MyST

Navigate to the same directory and activate the virtual environment again:

```bash
source venv/bin/activate
```

Start MyST:

```bash
myst start
```

You may be prompted to install Node.js at this step. Accept by pressing:

```text
y + Enter
```

Navigate to the URL displayed in the terminal (example):

```text
http://localhost:3000
```

Changes made to the content will automatically refresh the MyST website.

---

# Contribution Guide

Use Jupytext to sync (or create) Jupyter notebooks according to `jupytext.toml`.

```bash
jupytext --sync src/*.md
```

## Notebook Workflow

- Notebooks should be created and edited using JupyterLab (`.ipynb`)
- Notebooks should then be converted/synced to Markdown (`.md`)
- Only the `.md` version (displayed by MyST) should be committed to the repository

## Contributing Changes

To contribute a notebook or feature:

1. Fork the repository
2. Create a branch on your fork
3. Commit your changes to that branch
4. Open a pull request from your branch to this repository’s `main` branch
