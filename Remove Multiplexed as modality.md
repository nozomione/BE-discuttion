# Q&A: Removing Sample.Modalities.MULTIPLEXED

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
- [x] Take what is in github markdown file and place in this document (HackMD)
- [x] Create a space under each question for an answer
- [x] Look at where / how we use `Sample.Model.MULTIPLEXED` this might be covered in your original notes (see **Nozomi's Note** section below)
- From david: Prepare intro to loading of multiplexed samples

#### Nozomi's Note:
**Q:** Where/how `Sample.Modalities.MULTIPLEXED` is used? 

In the`Project` model:

**#1:** In `combine_multiplexed_metadata`, it's assigned to the attribute `modality` which is passed as an argument to methods to get the metadata information ([L288](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/project.py#L228)):

```python
def combine_multiplexed_metadata(...):
     modality = Sample.Modalities.MULTIPLEXED ## "MULTIPLEXED"
```

**#2:** In `get_metadata_field_names`, it's used as a key in `ordering` dictionary which is used for sorting coulmns in the output files ([L773](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/project.py#L773)):

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
**Q:** What is the method for viewing the tables (e.g., [`samples`](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/sample.py#L11)) in the databse? Do we use any GUI tools (e.g., [pgAdmin](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ConnectToPostgreSQLInstance.html#USER_ConnectToPostgreSQLInstance.pgAdmin))?

**A:**


Q: Where does the `modalities` attribute ([L109](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/sample.py#L109)) in the `Sample` model utilized in the codebase? The `Project` model defineds the `modalities` field ([L51](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/project.py#L51)). Does this field relate to the `modalities` attribute in the `Sample` model?

**A:**
