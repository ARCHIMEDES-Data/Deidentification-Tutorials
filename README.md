# Deidentification-Tutorials

This script-style Jupyter notebook introduces researchers to Python-based tools for reducing identifiability in DICOM data. It demonstrates common technical approaches used in de-identification workflows, with examples informed in part by HIPAA Safe Harbor concepts for identifiers commonly removed from health data.

It allows users to:
- Use either a sample or custom DICOM file
- Choose specific tags to clear, mask, generalize, pseudonymize, or suppress
- Filter and batch process files from a folder


#### NOTE:
This tutorial is provided for educational and research workflow support only.
**It does not provide legal advice, and it does not by itself determine whether data are de-identified, anonymized, or otherwise compliant under any specific law or institutional policy.**
Users are responsible for ensuring that their data handling practices meet the requirements of the laws, regulations, ethics approvals, data sharing agreements, and institutional policies applicable to their project.

**For Canadian users:**
Methods commonly used to satisfy HIPAA de-identification standards are not automatically sufficient for data to be considered anonymized or non-identifiable under Canadian laws. In many Canadian contexts, data that have undergone HIPAA-style de-identification may still be treated as coded or potentially identifiable. Consult your institution, privacy office, REB, or legal/privacy experts where appropriate.

## Install Guide (MyST)

#### In the terminal, navigate to the root directory of this project and create a virtual environment, activate it, then install the dependicies
```
python -m venv venv
source venv/bin/activate
pip install -e .
```

#### Run Jupyter Server (Optional. Enables ability to launch notebook from MyST)
```
jupyter lab --NotebookApp.token=0123456789abcdef0123456789abcdef0123456789abcdef --NotebookApp.allow_origin='http://localhost:3000'
```

The token's value can be changed, but must match the `token` field in `myst.yml`.  

The `allow_origin` uses the default MyST port, which will be confirmed later. This will work so long as another service is not bound to port 3000. Otherwise, make the command above use the correct port.  

Jupyter Server will print out which port it is running on. If it is not 8888, modify the `url` field in `myst.yml`.  

This terminal is now running JupyterLab and a new terminal should now be used to run MyST.

#### Navigate to the same directory and activate the virtual environment in this terminal:
```
source venv/bin/activate
```

#### Run MyST:
```
myst start
```

You may be prompted to install `node` at this step. Accept by pressing `y` then `Enter`.

Navigate to the URL displayed: `ex: http://localhost:3000`

Changes made to the content will trigger a refresh of the MyST website


## Contribution Guide

Use [Jupytext](https://jupytext.readthedocs.io/en/latest/) to sync (or create) your Jupyter notebooks according to [jupytext.toml](jupytext.toml).

```
jupytext --sync src/*.md
```

Notebooks should be created and edited using Jupyterlab (`.ipynb`) and converted to Markdown (`.md`).  
Only the `.md` version, which is the version MyST displays, should be committed.  


To commit a notebook or feature:
1. Fork the repository
2. Create a branch on your fork
3. Commit your feature to this branch
4. Make a pull request from your branch to this repository's `main` branch
