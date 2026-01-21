## Introduction
HR Recruiting system. <br>
The process include:
1. initialization
    + setup environment
    + setup data and permission tables in database
2. indexing
    + 2.1 parse - change any file into pure text file.
    + 2.2 extract - extract info to formatted json by llm.
    + 2.3 load - insert and update json into knowledge base
3. retriving
    + 3.1 login - authorization required
    + 3.2 query preprocess - query classification and decomposer
    + 3.3 retrieve - retrieve data by query
    + 3.4 answer - answer the query by retrieved content
4. evaluation
    + 4.1 dataset generation
    + 4.2 evaluation system


## Environment
+ Docker structure:
    + reference `docker-compose.yaml`

    | containers   | function      | implementation | mount folder          |
    | -            | -             | -              | -                     |
    | localllm     | local LLM     | Ollama         | model                 |
    | relationaldb | relational db | PostgreSQL     | sys                   | 
    | vectordb     | vector db     | QDrant         | sys                   |
    | python       | all scripts   | python         | current + resume data |

+ File structure
    + cfgs/ ------> all config (include database schema)
    + hrrecruit/ ------> main package
        + common/
        + db_interface/ ------> sql & qdrant 
        + prompts/
        + index/
            + parse/
            + extract/
            + load/
        + retrieve/
            + login.py
            + perprocess.py ------> query classification and decomposer
            + retrieve ------> core algorithm
            + answer.py ------> llm answer with retrieve
        + async_func.py
        + llm_api.py
    + scripts/ ------> **Main Entrypoint**
        + main/
            + index.py
            + retrieve.py
        + databases/
            + build.py
            + show.py
            + clear.py
    + tests/ ------> act as both "test" and "how to use"
        + hrrecruit/ ------> same architecture as `hrrecruit/` for pytest
        + input_data/ ------> example data for test inputs
        + output_results/ ------> test outputs
        + expect_results/ ------> expected Test outputs


## Data schema

#### relational database - data part
see `/app/cfgs/db_schema.csv`

| id | table name       | introduction | storage | records<br>per candidate |
| -  | -                | -            | -       | -                        |
| 1  | basic_info       | like name, contact info, apply info, etc. | normal        | 1 |
| 2  | channel          | how to know this job, most are bool       | normal        | 1 |
| 3  | work_experiences | like time, company, job title salary, etc.| normal        | multiple |
| 4  | educatuions      | like time, school, department, etc.       | normal        | multiple |
| 5  | languages        | language abilities                        | normal        | multiple |
| 6  | hr_labels        | folder (candidate) name and custom tags   | hashtags      | multiple |
| 7  | basic_info_sum   | summarization of basic_info               | with vectordb | 1 |
| 8  | work_experiences_sum | summarization of work_experiences     | with vectordb | multiple |
| 9  | educations_sum       | summarization of educations           | with vectordb | multiple |
| 10 | languages_sum        | summarization of languages            | with vectordb | multiple |
| 11 | project_skills_sum   | summarization of work_experiences     | with vectordb | 5 |
| 12 | personality_test     | summarization of work_experiences     | with vectordb | 1 |
| 13 | interview_record     | summarization of work_experiences     | with vectordb | 1 |

+ hashtags utilize GIN index in postgreSQL
+ summarization text and text_id is stored in sql.
+ vectordb metadata does not store text, use text_id to load text in sql instead.  


#### relational database - permission part

1. permission_cols

| attr_id | table_name | column_name   |
| -       | -          | -             |
| 1       | basic_info | first_name_zh |

2. permission_def

| permission_id | permission_name | attr_1 | attr_2 |
| -             | -               | -      | -      |
| 2             | normal          | true   | false  |

3. permission_accounts

| user_id | name  | account | password | permission_root | permission_normal |
| -       | -     | -       | -        | -               | -                 |
| 1       | admin | admin   | 8888     | true            | true              |
| 2       | abc   | abc     | 1234     | false           | true              |            


#### vector database

+ collections will be built before insertion,

+ schema see `/app/cfgs/cfg.yaml` <br>
    load.vectordb_args.collection_name_list
    - basic_info_sum
    - work_experiences_sum
    - educations_sum
    - languages_sum
    - projects_skills_sum
    - personality_test
    - interview_records

+ each meta data is like
```json
payload = {"resume_id": resume_id, "text_id": text_id}
```

+ define query unit:
```python
@dataclass
class Point:
    vector_id: str
    score: float
    payload: Dict
```


## Implementation details

1. initialization
2. indexing
    + 2.1 parse
        + change any file into formatted data **(.txt files)**
        + support format include:
            + `.pdf`
                + use pymupdf to extract text
                + if text is not enough in certain pages, perform teserract ocr to those pages 
            + `.docx` and `.doc`
                + convert to pdf and use pdf pipeline
            + `.txt` and `.csv`
                + extract text directly
            + `.xlsx` and `.xls`
                + convert to string
            + `.jpg`, `.jpeg`, `.png`
                + perform ocr as in pdf
            + directory
                + recursively parse files and flatten
                + same file name but in different subfolder will be **overwritten**. 
            + `.zip`, `.7z`
                + auto decompressed and use directory pipeline
        + available ocr module
            + teserract ocr: en, sim-ch, tra-ch
    
    + 2.2 extract
        + change all txt into formatted **json**
            + example output is in `/app/hr_recruit/outputs/exp1/json/*.json`
            + the type of formatted json are named in <br>
            `/app/cfgs/db_schema.csv` table_io_format column <br>
            the type includes <br>

        ```python
        # example output of extract:
        # the outest key must be same as table name
        {
            # type 1 - normal
            "basic_info": {
                "first_name_en": "james",
                "last_name_en": "chao",
                ...
            },
            ...

            # type 2 - normal list (inner key must be same as outer key)
            "educations": {
                "educations": [
                    {
                        "school": "國立xx大學",
                        "deparment": "yyy",
                        ...
                    },
                    ...
                ]
            },
            ...

            # type 3 - hashtags (inner key must be "tags")
            "hr_labels": {
                "tags": ["tag1", "tag2"]
            }

            # type 4 - embedding (inner key must be "descriptions")
            "educations_sum": {
                "descriptions": [
                    "高中: 國立xx高級中學, 普通科, 2007年9月 - 2010年6月, 已畢業",
                    "大學: ..."
                ]
            }
        }
        ```

        + some files can be **direct processed** <br>
            the keyword is define in `/app/cfgs/cfg.yaml`
            + personality test: ["職能性格"]
            + interview record: ["面談紀錄"]
            + hr_labels: ["hr_labels"]
        + other files **extract by llm after concatenate** <br>
            see full content in `/app/hrrecruit/prompts/extract/`<br>
            all prompts include
            + p1_basic_info_normal.yaml
            + p2_basic_info_application.yaml
            + p3_basic_info_sensitive.yaml
            + p4_channel.yaml
            + p5_work_experiences.yaml
            + p6_educations.yaml
            + p7_languages.yaml
            + p8_basic_info_sum.yaml
            + p9_work_experiences_sum.yaml
            + p10_educations_sum.yaml
            + p11_languages_sum.yaml
            + p12_project_skills_sum.yaml
        
    + 2.3 load
        + Put formatted json into knowledge base
        + Must identify whether the record is already in the knowledge base **(deduplication)**
            + the logic is in `/app/hrrecruit/load/is_exist_logic.py`
        + Depends on above 4 types of json, each json has its own upsert method

3. retrieve
    + 3.1 login
        + if not authorized, quit with error message
        + get permitted columns: Dict[str, List[str]]
            ```python
            permitted_cols = {
                "basic_info": ["first_name_zh", "last_name_zh", ...],
                "educations": [...]
            }
            ```
    + 3.2 preprocess
        + query classification
            + 1: retrieve ownly
                + mainly for group based question, higher top-k
                + e.g. "幫我找出有...經驗的人才"
            + 2: retrieve and answer
                + mainly for candidate based question, lower top-k
                + e.g. "xxx的工作經歷有哪些"
            + 3: irrelevant
                + quit with error message
        + query decomposer <br>
            split to each domain, each has 1 or None of queries
            + basic_info
            + work_experiences
            + educations
            + languages
            + projects_skills
            + personality_test
            + interview_record
        + negative query splitter
            + split each query into postive part and negative part
            ```python
            subquery_dict = {
                "positive": {
                    "work_experiences": "有xxx的經驗"
                },
                "negative": {
                    "work_experiences": "沒有在xxx公司工作過"
                }
            }
            ```
        + range splitter
    + 3.3 retrieve
        + hybrid algorithm
            + each algorithm weight is define in `/app/cfgs/cfg.yaml`
            + algorithm contains
                + semantic positive
                + semantic negative
                + hashtag positive
                + hashtag negative
            + the outcome is score dict, where key is resume_id and val is score
                ```python
                score_dict = {"abc123": 0.85, "def789": 0.56, ...}
                ```
        + resume extraction
            + rank the top-k resume id
            + extract all sql data excluding summarization portion (for retrieve only)
            + mask the non-permitted column with "因權限不足而無法顯示"
        + return list of resume: List[Dict]
    + 3.4 answer
        + formatted the list of resume into **csv** for better comparison
        + call llm if query is in "retrieve and answer" class

4. evaluation


## Quick start

Take the current data `data/08_AI人員共同工作環境使用之測試區/觀音` format as example <br>
Each candidate has its own directory to store their personal document<br>


1. build python docker
```bash
cd python
docker build -t hrrecruit:v0.0.7 .
cd ..
```

2. launch container
```bash
docker compose up -d
```
+ the command `docker compose down` closes all containers
+ the data will be mount to `/data` in the python container 

3. login python container
```bash
bash env.sh
```
+ docker building step has already installed all 3rd-party packages (exclude self package: hrrecruit)
+ login step install the self package: hrrecruit only (without internet, with editable mode)

4. Set up relational db
```bash
poetry run python /app/scripts/databases/build.py
```
+ db schema is at `cfgs/db_schema.csv`
+ if db exists already, nothing will happen
+ more additional scripts for developer
    + drop all data or tables: `/app/scripts/databases/clear.py`
    + show info of database `/app/scripts/databases/show.py`

5. run indexing script
```bash
poetry run python /app/scripts/main/index.py
```

6. run retrieve script
```bash
poetry run python /app/scripts/main/retrieve.py
```
+ retrieve query is in the file


## Some stress test
    + Current max data:
        + 田綺綺 content contains thesis, pasred txt: total character length 111k, token size 40k
        + most people, parsed txt: total character length <25k, token size <10k


## Todo list
2. retrieve query decomposer
3. retrieve by given resume id 
4. permission
