## Introduction
This is an interview project in 8 hours.


## Core Design
1. Extract pdf information and store chunks by `Pymupdf` and `Faiss`.
2. Use `regex` and `faiss search` extraction accompany with `LLM` to extract key information


## Quick start
1. Launch docker container
+ windows
```powershell
.\test.ps1
```
+ linux
```bash
bash ./test.sh
```


2. Install dependencies
```bash
pip install -r requirements.txt
```


3. Run indexing script
```bash
python extract_pdf_store_chunks.py
```
You will see `processed_data/` includes faiss indices and chunks in csv format.


4. Run retrieving script
```bash
python query_for_answer.py
```
And finally you will see the extracted key information in
`processed_data/retrieved_info.json`


## Example of the outputs
```json
{
    "name": "Arthur Miller",
    "ssn": "",
    "claim_title": "",
    "dli": "",
    "aod": "",
    "date_of_birth": "",
    "age_at_aod": 0,
    "current_age": 60,
    "last_grade_completed": 0,
    "attended_special_ed_classes": "",
    "medical_events": [
        {
            "date": "02-18-2023",
            "provider": "METRO HEALTH & WELLNESS CENTER MER",
            "reason": "Initial",
            "page": 19
        },
        {
            "date": "02-09-2023",
            "provider": "METRO HEALTH & WELLNESS CENTER MER",
            "reason": "Initial",
            "page": 19
        },
        {
            "date": "01-01-2020",
            "provider": "Dr. Elias Vance",
            "reason": "Office/Clinic/Outpatient visit",
            "page": 195
        },
        {
            "date": "12-28-2022",
            "provider": "Unknown",
            "reason": "Medical Opinion: No",
            "page": 6
        },
        {
            "date": "02-06-2023",
            "provider": "",
            "reason": "Medical Opinion No",
            "page": 5
        },
        {
            "date": "06-01-2020",
            "provider": "Primary Care Physician",
            "reason": "Evaluation and management of back, neck, and hip pain (regular evaluations, medication scripts, routine testing, specialist referrals, and diagnostic testing).",
            "page": 196
        },
        {
            "date": "07-07-2023",
            "provider": "Sterling Health Clinic",
            "reason": "Will email SAM/ASC to review.",
            "page": 280
        },
        {
            "date": "04-08-2024",
            "provider": "Central Plains Medical Center",
            "reason": "Email from MediRecords Solutions Inc. (ereq #188937708) \u2013 sent NO RECS statement to SAM/MR backup folders and will email Datavant about adjustment.",
            "page": 280
        },
        {
            "date": "04-09-2024",
            "provider": "Metro Health & Wellness Center",
            "reason": "Email from MediRecords Solutions Inc. (ereq #188937318) \u2013 sent NO RECS statement to SAM/MR backup folders and will email Datavant about adjustment.",
            "page": 280
        },
        {
            "date": "04-15-2024",
            "provider": "Anya Sharma, NP",
            "reason": "Email from Shannon J (Datavant) \u2013 invoice submitted.",
            "page": 280
        },
        {
            "date": "04-13-2023",
            "provider": "Medical Source",
            "reason": "14",
            "page": 2
        },
        {
            "date": "04-20-2023",
            "provider": "METRO WELLNESS CLINIC",
            "reason": "12",
            "page": 2
        },
        {
            "date": "07-21-2021",
            "provider": "Sterling Health Clinic",
            "reason": "34",
            "page": 2
        },
        {
            "date": "02-26-2024",
            "provider": "Unity Care Clinic",
            "reason": "3",
            "page": 2
        },
        {
            "date": "07-06-2023",
            "provider": "Sterling Health Clinic",
            "reason": "16",
            "page": 2
        },
        {
            "date": "07-19-2021",
            "provider": "Sarah Chen (Referring Provider)",
            "reason": "Lumbar Spine MRI requested (ordered for other reason \u2013 pain with radiation)",
            "page": 461
        },
        {
            "date": "06-29-2021",
            "provider": "",
            "reason": "Patient seen with complaint of pain with radiation",
            "page": 461
        },
        {
            "date": "08-11-2023",
            "provider": "6145550102",
            "reason": "Basic review of endocrine, hematologic, and allergic/immune systems \u2013 negative findings",
            "page": 451
        },
        {
            "date": "08-15-2022",
            "provider": "",
            "reason": "Asked about medication use (prescription or non-prescription)",
            "page": 194
        }
    ],
    "SSN": "456-12-7890",
    "AOD": [
        "02-08-2023"
    ],
    "DOB": "05-21-1965"
}
```

## Remarks
+ since data is not huge and not scaled currently, FAISS is a good choice
+ long answer can properly be queried from FAISS such as medical events 
+ short answer is hard to queries, use regex for the help
+ some short answer without specified format (e.g. Title) is challenging.
+ Since the limited time, the final part of the code has not well-modularized (but also workable), which is possible to be optimized in the future work. Keys of the directions include readibility, reusability and scalability
+ The final parts could include json to docx by `python-docx`.
