# Q&A

**Epic:** [Remove Sample.Modalities.MULTIPLEXED](https://github.com/AlexsLemonade/scpca-portal/issues/695) 

**Merged PR for the computed file:** [Multiplexed is now SingleCell modality](https://github.com/AlexsLemonade/scpca-portal/pull/672)

During our weekly "Nozomi + David BE: Discussion" meeting, we use this document to keep track of the questions (asked by Nozomi) and answers (provided by David). This information is then utilized to plan the implementation strategies and file issues for the epic.

This also includes a homework (to-do list) section that we aim to complete before our next meeting.

(The blob is included for future reference.)
## 05/08/2024
**Q:** Why does the [`Sample`](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/sample.py#L34)  model redefine the `has_multiplexed_data` field while inheriting from [`CommonDataAttributes`](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/base.py#L12), which already defines that field? What differentiates them and why?

```python
has_multiplexed_data = models.BooleanField(default=False)
```

**A:** 
- Short Answer: This is a bug
- Longer answer: This is a potential bug. what is happening here is that the sample inherits the property on the class and then redefines it in the same way.

JS Analogy:
```javascript
const CDA = {attribute: "value"}
const Sample = {...CDA, attribute: "value"}
```

---

**Q:** Why is `has_multiplexed_data` assigned differently at the project and sample levels?

**A:**
- project will be true if 1 sample is multiplexed
- sample will be true if it was multiplexed

---

**Q:** Where does the value of `multiplexed_with` come from?

In the `Sample` model, the `multiplexed_with` field is defined ([L40](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/sample.py#L40)):

```python
multiplexed_with = ArrayField(models.TextField(), default=list)
```

Then its value is assigned to `has_multiplexed_data` ([L59](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/sample.py#L59)):

```python
has_multiplexed_data = bool(data.get("multiplexed_with"))
```

**A:** 
- when we load data, we first download all of the data and metadata from s3, that data is downloaded with a folder structure that is `project_id/sample_id/data.file_ext`
- some sample_id folder names will actually be `project_id/sample_id1,sample_id2,sample_id3/data.file_ext`
- we know that this is a multiplexed sample
- sample_id1.multiplexed_with = [sample_id2, sample_id3]

### Homework
- [x] Look up: [List Comprehension w/ if](https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions)
  - Additionally, read [Nested List Comprehensions](https://docs.python.org/3/tutorial/datastructures.html#nested-list-comprehensions)
- [ ] Look up: Class methods
- [x] Take what is in github markdown file and place in this document
- [x] Create a space under each question for an answer
- [x] Look at where / how we use `Sample.Model.MULTIPLEXED` this might be covered in your original notes (see **Nozomi's Note** section below)
- From david: Prepare intro to loading of multiplexed samples
   - This was covered in Engerring Meeting [2024-05-14 - ScPCA Portal Future Thoughts](https://data-lab-knowledge.slab.com/posts/2024-05-14-sc-pca-portal-future-thoughts-tdp2h14j)

#### Nozomi's Note:
**Q:** Where/how `Sample.Modalities.MULTIPLEXED` is used? 

In the`Project` model:

**#1:** In `combine_multiplexed_metadata`, it's assigned to the attribute `modality` which is passed as an argument to methods to get the metadata information ([L288](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/project.py#L228)):

```python
def combine_multiplexed_metadata(...):
     modality = Sample.Modalities.MULTIPLEXED ## "MULTIPLEXED"
```

**#2:** In `get_metadata_field_names`, it's used as a key in the `ordering` dictionary which is used for sorting coulmns in the output files ([L773](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/project.py#L773)):

```python
 def get_metadata_field_names(self, columns, modality):
        ordering = {
            Sample.Modalities.MULTIPLEXED: (...), 
            # a tulpe of metadata field names for multiplexed libraries
```


In the `Sample` model:

**#1:** In the `Modalities` class, `MULTIPLEXED` is added as the modality, and to `NAME_MAPPING ` ([L15](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L15)):

```python
class Modalities:
      MULTIPLEXED = "MULTIPLEXED"

      NAME_MAPPING = {
         MULTIPLEXED: "Multiplexed", 
     }
```

**#2:** In `get_output_metadata_file_path`, it's used as a key in the returned dictionary ([L100](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L100)):
```python
 def get_output_metadata_file_path(scpca_sample_id, modality):
        return {
            Sample.Modalities.MULTIPLEXED: common.OUTPUT_DATA_PATH
            / f"{scpca_sample_id}_multiplexed_metadata.tsv",
```

This method is called in `output_multiplexed_metadata_file_path` to get the path of the metadata file for multiplexed ([L142](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L142)):

```python
def output_multiplexed_metadata_file_path(self):
 return Sample.get_output_metadata_file_path(self.scpca_id, Sample.Modalities.MULTIPLEXED)
 # e.g., "/home/user/code/SCPCS000133_multiplexed_metadata.tsv" in local
```

**#3:** In the property `modalities`, it's assigned as the value of the key `has_multiplexed_data` in `attr_name_modality_mapping` ([L113](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L113)):
```python
  def modalities(self):
        attr_name_modality_mapping = {
            "has_multiplexed_data": Sample.Modalities.MULTIPLEXED,
        }
        return sorted(
          [
            # e.g., 'MULTIPLEXED'
            Sample.Modalities.NAME_MAPPING[modality_name] 
            # e.g., [('has_multiplexed_data', 'MULTIPLEXED')]
            for attr_name, modality_name in attr_name_modality_mapping.items() 
            # e.g., if self.has_multiplexed_data is true
            if getattr(self, attr_name) 
           ]
    # e.g., Returns a sorted list of available modalities 
    # ['Bulk RNA-seq', 'Multiplexed' ]
```


## 05/15/2024
**Q:** What is the method for viewing the tables (e.g., [`samples`](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/sample.py#L11)) in the databse? Do we use any GUI tools during development (e.g., [pgAdmin](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ConnectToPostgreSQLInstance.html#USER_ConnectToPostgreSQLInstance.pgAdmin))?

**A:** 

Short Answer: No we don't really use GUI tools. There are various ways to connect to the database but for the security purposes, we don't typically do this. It is possible to SSHing to the server but we don't encourage it in our workflow. 

Then how do we get a better understanding of the database?

The ways to accomplish:
We have two `sportal` commands that allow you to look at what is in the database:
1. `sportal shell` which will load up a python shell in the context of your application. Where you can import a model and then use the ORM to query rows from that model table. You will get back a `queryset` but this can be converted to model instances.

e.g. )
```python
sportal shell
# python interactive shell 
>>> from scpca_portal.models.project import Project
>>> p = Project.objects.all() # e.g., <QuerySet [<Project: Project SCPCP000001>, <Project: Project SCPCP000002>, <Project: Project SCPCP000003>]>
>>> p.filter(scpca_id='SCPCP000001') # <QuerySet [<Project: Project SCPCP000001>]>
>>> p.first() # <Project: Project SCPCP000001>
```

2. `sportal postgres-cli` which will give you access to the postgres shell where you can run sql query direction against your database.

e.g. )
```bash 
sportal postgres-cli
# postgres interactive terminal 
postgres=# \l
postgres=# select * from sample
postgres=# \q
```

**Q:** Where is the `modalities` attribute ([L109](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/sample.py#L109)) in the `Sample` model utilized in the codebase? 

```python
@property
    def modalities(self):
```

The `Project` model defineds the `modalities` field ([L51](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/project.py#L51)). Does this field relate to the `modalities` attribute in the `Sample` model?

```python
 modalities = ArrayField(models.TextField(), default=list)
```

**A:**

It is used when updating counts on projects, these are aggregate values that describe the project. So every sample has modalities that are evaluated via the `modalities` property. Then when we update a project, we call `project.update_counts` and in that method we use this `sample.modalities` property to collect all of the project's samples and save them to the database for that project.

`Sample.modalities` are evaluated based on properties at runtime against a sample instance.

e.g. )

Sample 1:
 has_cite_seq = True
 modalites is evaluated to "Cite-seq"
 
Sample 2:
 has_bulk_rna = True
 modalites is evaluated to "Bulk rna"

The project is updated at load_data time. And then stored to the database. So it basically just checks all of the project's samples modalities and adds them to a set (to remove duplicates) and then saves that to the `Project` model.

### Homework
- [ ] Look up: Python Lambda function
- [ ] Look up: ORM/Query API (e.g., for `sportal shell`)
- [ ] Go over ScPCA test commands/files 
- [ ] Review the usage of `Sample.modalities` (for Q2)
- [x] Based on our chat, outline the steps to simplify the column sorting for the metadata TSVs (related issue [#699](https://github.com/AlexsLemonade/scpca-portal/issues/699)) before drafting an issue for it (see **Nozomi's Note** section below)
  - Filed issue: [Add util function for sorting columns of metadata TSVs](https://github.com/AlexsLemonade/scpca-portal/issues/703)

#### Nozomi's Note:
Currently, we sort the columns of metadata TSV files during the write operations using the `get_metadata_field_names` method ([L770](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/project.py#L770)) in the `Project` model:

```python
  def get_metadata_field_names(self, columns, modality):
      ordering = {
            Sample.Modalities.MULTIPLEXED: (
                "scpca_sample_id",
                "scpca_library_id",
                ...
```

Based on the modality, this method returns the appropriate metadata field names for each TVS file. However, this logic will need to be updated as we're removing `MULTIPLEXED` from the modalities.

Furthermore, we're aiming to further simplify this logic by setting a global sort order for all output metadata files and making it modality-independent.

These are the steps for simplifying the colum sorting for the metadata TSVs: 

**Step 1:** Add the global sort order list to the project setting (`/config/common.py`)

**Step 2:** Create a new utility function that:
- sorts the set of keys returned by utils.get_keys_from_dicts based on the global sort order
- returns that sorted keys as a list

**Step 3:** Replace `get_metadata_field_names` with this newly created utility function

More details: [2024-05-21 - 2024-06-03](https://data-lab-knowledge.slab.com/posts/2024-05-21-2024-06-03-iy7h83m0)

## 05/22/2024
#### General:

**Q:** Is the property `multiplexed_computed_file` in the `Project` ([L97](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/project.py#L97)) and `Sample` ([L193](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/sample.py#L193)) models used solely for testing for `load_data` (e.g., [L365](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/test/management/commands/test_load_data.py#L365))? 


```py
@property
    def multiplexed_computed_file(self):
        try:
            return self.project_computed_files.get(
                modality=ComputedFile.OutputFileModalities.SINGLE_CELL,
                format=ComputedFile.OutputFileFormats.SINGLE_CELL_EXPERIMENT,
                has_multiplexed_data=True,
            )
        except ComputedFile.DoesNotExist:
            pass
```
Is this a common pattern in Django to add properties for testing in the model? If so, should we add comments to describe test properties? 

Or considering the frequent changes in the codebase, will it eventully be refactored?

**NOTE:** The same question applies to the following properties in the models:
- `single_cell_computed_file` in `Project` ([L144](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/project.py#L144)) and `Sample` ([L204](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/sample.py#L204)).
- `single_cell_anndata_computed_file` in  ([L166](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/project.py#L166)) and `Sample` ([L215](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/sample.py#L215)).
- `single_cell_merged_computed_file` in `Project` ([L155](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/project.py#L155))
- `single_cell_anndata_merged_computed_file` in `Project` ([L177](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/project.py#L177))
- `spatial_computed_file` in `Project` ([L188](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/project.py#L188)) and `Sample` ([L226](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/sample.py#L226))

**A:** 

This is just a helper not specific for testing. We can use it in our application because it is defined on the model.

<hr />

**Q:** Currently, different but similar methods are defined for creating README files in the `Project` model. If this is required, what are the advantages of this pattern over adding a single method (e.g., whitn the model, in `utils`) to handle the generation of all readme files?

e.g. ) Creating the readme file for Single-cell ([L236](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/project.py#L236)), Spatial ([L281](https://github.com/AlexsLemonade/scpca-portal/blob/f6ec0f77e2e020fe4bb5221daf6a6439bfa782a9/api/scpca_portal/models/project.py#L281)):

```py
# For single-cell
def create_single_cell_readme_file(self):
        """Creates a single cell metadata README file."""
        with open(ComputedFile.README_SINGLE_CELL_FILE_PATH, "w") as readme_file:
            readme_file.write(
                render_to_string(
                    ComputedFile.README_TEMPLATE_SINGLE_CELL_FILE_PATH,
                    context={
                        "additional_terms": self.get_additional_terms(),
                        "date": utils.get_today_string(),
                        "project_accession": self.scpca_id,
                        "project_url": self.url,
                    },
                ).strip()
            )

# For Spatial
 def create_spatial_readme_file(self):
        """Creates a spatial metadata README file."""
        with open(ComputedFile.README_SPATIAL_FILE_PATH, "w") as readme_file:
            readme_file.write(
                render_to_string(
                    ComputedFile.README_TEMPLATE_SPATIAL_FILE_PATH,
                    context={
                        "additional_terms": self.get_additional_terms(),
                        "date": utils.get_today_string(),
                        "project_accession": self.scpca_id,
                        "project_url": self.url,
                    },
                ).strip()
            )

```

**A:** 

We are moving from preparing files before compilation of zipped computed files to creating files at time of write.

Before, we were writing files to the output folder that would be used to populate the zip files.
```
before:
data/
- input
    - project_metadata.tsv
    - SCPCP000001/
    - ... rest of project folders ...
- output
    - readme_anndata_scpcp000001.tsv // will go into a zip but renamed
    - anndata_sample_metadata_scpcp000001.tsv
    - anndata_scpcp000001.zip (contains anndata_readme, anndata_metadata)
```

Now we want to only write the files that will be uploaded to the output folder.
```
after:
data/
- input
    - project_metadata.tsv
    - SCPCP000001/
    - ... rest of project folders ...
- output
    - scpcp000001.zip // only the zip files
```

<hr />

#### Devlopment / Testing:
During the last meeting, attempting to load the project data locally on my work machine resulted in the `InvalidAccessKeyId` error (also not enough disk space on this machine to load data via the `sportal load-data` command):

e.g. ) Trying to load the project `SCPCP000001`:

![Screenshot 2024-05-22 at 1 47 13 PM](https://github.com/nozomione/BE-discuttion/assets/31800566/46337ec6-ecb4-4b53-bfdb-e90f4f7903e3)


**Q:** What is the workaround/approach to develop and test my implementation during development if no data can't be loaded locally? (more specifically, when working on the ticket [Add util function for sorting columns of metadata TSVs](https://github.com/AlexsLemonade/scpca-portal/issues/703)) 


**A:** 

You can run the tests for this for a couple reasons:
- You don't want to run against an actual project as that would take a long time.
- We currently are changing how we provide permissions to access lab resources so there will be tooling for you to run "integration " tests in the future.

You want tests to fail for the changing of metadata keys because we are changing the conditions for the test to pass.
<hr />

**Q:** The local testing can be easily done on FE development with JavaScript via a browser's console etc. Is there a similar tool that can be incorporated in BE development (e.g., for Python/Django)?

**A:**

You want to write a unit test and call it via:
- `sportal test-api scpca_portal.test.test_utils.NameofYourTestClass`
- use `sportal shell` if you need to access a model in the application
- use `python3` if you want to just validate some python behavior 

#### Nozomi's Note:
##### Before the meeting:
- Based on the feedback left in the issue, updated the Solution or next step section in [Add util function for sorting columns of metadata TSVs](https://github.com/AlexsLemonade/scpca-portal/issues/703).
  
##### After the meeting:
In addition, we covered the following topics:

**Topic 1.** ScPCA Commands 

For migrations:
- [`sportal showmigrations`](https://github.com/AlexsLemonade/scpca-portal/blob/80d21554975db650bf5aca4cd9c6d4691ac84cc9/bin/sportal#L30): lists all the migrations available in the project
- [`sportal migrate`](https://github.com/AlexsLemonade/scpca-portal/blob/80d21554975db650bf5aca4cd9c6d4691ac84cc9/bin/sportal#L29): syncs the database state with the latest migrations 

e.g.) To use the above commands to selectively apply the named migration to the databse:
```python
# Step 1: list available migrations and select a migration name
sportal showmigrations # e.g., 0039_computedfile_includes_celltype_report 

# Step 2: pass the name to migrate command along with the app label
# migrate [app_label] [migration_name]
sportal migrate scpca_portal 0039_computedfile_includes_celltype_report
```

> [!note]
> To prevent bugs, make sure to `sportal down` the running server before migration. 

For the test input bucket:

Alternatively, you can load a project from the publicly accessible test input bucket onto the local server by passing the argument [`--input-bucket-name`](https://github.com/AlexsLemonade/scpca-portal/blob/80d21554975db650bf5aca4cd9c6d4691ac84cc9/api/scpca_portal/management/commands/load_data.py#L122):

```
sportal load-data --input-bucket-name <INPUT_BUCKET_NAME> --scpca-project-id <PROJECT_ID>
```

> [!note]
> To prevent an error, make sure to white list `scpca` in [`ALLOWED_SUBMITTERS`](https://github.com/AlexsLemonade/scpca-portal/blob/80d21554975db650bf5aca4cd9c6d4691ac84cc9/api/scpca_portal/management/commands/load_data.py#L18) when loading the test input bucket, but do not commit it.

**Topic 2.** Deleting Local API Data
 To clean up the API data on your machine (if needed):
 
```
 sudo rm -r api/volumes_postgres/volumes_postgres/
```

### Homework
- Continue to work on the unchecked HW items from the previous sessions
- Work on [Add util function for sorting columns of metadata TSVs](https://github.com/AlexsLemonade/scpca-portal/issues/703)

## 05/29/2024
We went over the opened PR [703 - Add util function for sorting columns of metadata TSVs](https://github.com/AlexsLemonade/scpca-portal/pull/727) for reviews and also covered the following items: 

#### Check the content of output files:

1. Comment out the following line in `test_load_data`:

```py
class TestLoadData(TransactionTestCase):
    ...
    @classmethod
    def tearDownClass(cls):
        super().tearDownClass()
        # shutil.rmtree(common.OUTPUT_DATA_PATH, ignore_errors=True) <- this line
```

2.  Go to the folder `api/test_data/output` in your finder or `cd` into it via terminal and open a target:

```py
$ cd api/test_data/output && ll
```

e.g.) select the zip and open it
```py
-rw-r--r--  1 nozomi  staff   9.4K May 29 17:52 SCPCP999991_multiplexed.zip
-rw-r--r--  1 nozomi  staff   2.4K May 29 17:52 SCPCP999991_multiplexed_metadata.tsv
-rw-r--r--  1 nozomi  staff   7.9K May 29 17:52 SCPCS999992_SCPCS999993_multiplexed.zip # Open this file 
-rw-r--r--  1 nozomi  staff   1.9K May 29 17:52 SCPCS999992_multiplexed_metadata.tsv
-rw-r--r--  1 nozomi  staff   1.9K May 29 17:52 SCPCS999993_multiplexed_metadata.ts
```

```py
$ open SCPCS999992_SCPCS999993_multiplexed.zip
```

#### Use the [`pprint`](https://docs.python.org/3/library/pprint.html) module for readability:

```py
chars = ['a', 'b', 'c']
print(letters)

# Output:
['a', 'b', 'c']

from pprintimport pprint
pprint(chars)

# Output:
['a',
 'b',
 'c']

```

#### Use the [`maxDiff`](https://docs.python.org/3/library/unittest.html#unittest.TestCase.maxDiff) attribute for differences
We can highlight a difference by setting the `maxDiff` attribute value to `None` when the expected and generated values differ.

e.g.) 
```py
class Test(TestCase):
   self.maxDiff = None # Here
   # e.g.) for better readability, we may also print the file path
   print(f"sample_zip_path ========= {sample_zip_path}")
```

**Output:**

```py
[  'scpca_project_id',
   'scpca_sample_id',
   'submitter',
+  'sample_cell_count_estimate',
   'sample_cell_estimates',
   ...
+  'filtered_cell_count',
   ...
]
```

### Homework
- Continue to work on the unchecked HW items from the previous sessions

## 06/05/2024
**Q:** Would it be beneficial to give the [`load-data`](https://github.com/AlexsLemonade/scpca-portal/blob/825a4490383fc6c5ebe6a7a2de154cb4294d90be/api/scpca_portal/management/commands/load_data.py#L120) management command for users additional [`help`](https://docs.python.org/3/library/argparse.html#help) arguments? (This is more like a suggestion)?

e.g.)
```python=
parser.add_argument("--reload-all", action="store_true", default=False)
```

**A:** Not required at this time, since it's only being used internally for development. For the command reference, README is available.

**Q:** `action=argparse.BooleanOptionalAction`

> The BooleanOptionalAction is available in argparse and adds support for boolean actions such as ``--foo` and ``--no-foo` 
> 
> via [Python Doc: argparse](https://docs.python.org/3/library/argparse.html) (supprted in v3.9+)

e.g.)
```python
# First argument
parser.add_argument("--reload-all", action="store_true", default=False)

# Second argument
parser.add_argument(
  "--update-s3", action=BooleanOptionalAction, default=settings.UPDATE_S3_DATA
)
```

**A:** We take advantage of this built-in action to support boolean actions out of the box for the second argument. 

### Homework
**Issue:** [Management command to generate portal wide metadata only download](https://github.com/AlexsLemonade/scpca-portal/issues/708)

We went over the high-level overview of the linked issue above. 

Based on the `Library` model (upcoming changes), we'll implement a management command to handle the creation of a portal wide metadata file and its README.


By the next meeting:
- Create a sample management command to run in terminal for practice:
  - Step 1: Create a `x` management command
  - Step 2: Create a test and validate the output
- Outline individual steps for the linked issue based on our meeting conversation
