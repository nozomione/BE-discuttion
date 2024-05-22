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
- sorts the set of keys returned by `utils.get_keys_from_dicts` based on the global sort order
- returns that sorted keys as a list

**Step 3:** Replace `Project::get_metadata_field_names` with this newly created utility function

More details: [2024-05-21 - 2024-06-03](https://data-lab-knowledge.slab.com/posts/2024-05-21-2024-06-03-iy7h83m0)

#### Update:
Filed issue: [Add util function for sorting columns of metadata TSVs](https://github.com/AlexsLemonade/scpca-portal/issues/703) 

## 05/22/2024
