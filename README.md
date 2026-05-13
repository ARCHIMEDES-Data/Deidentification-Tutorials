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
