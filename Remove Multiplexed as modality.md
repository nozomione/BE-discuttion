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
- [ ] From david: Prepare intro to loading of multiplexed samples
- [x] Look at where / how we use `Sample.Model.MULTIPLEXED` this might be covered in your original notes
  - This section will be added to the next meeting note

## 05/15/2024


### Previous Homework

**Q:** Where/how `Sample.Modalities.MULTIPLEXED` is used?

At the`Project` model:

1. In the `combine_multiplexed_metadata` method, it is assigned as the value of the `modality` attribute ([L288](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/project.py#L228)):

```py
def combine_multiplexed_metadata(...):
     ...
     modality = Sample.Modalities.MULTIPLEXED ## "MULTIPLEXED"
```

2. In the `get_metadata_field_names(self, columns, modality)`, it is used as a key in the `ordering` dictionary whose value is a tuple of the metadata filed names for multiplexed libraries ([L773](https://github.com/AlexsLemonade/scpca-portal/blob/026a204a0aa89e6e2572038a46cbe154af7efbef/api/scpca_portal/models/project.py#L773))

```py
 def get_metadata_field_names(self, columns, modality):
        ordering = {
            Sample.Modalities.MULTIPLEXED: (...),
```


At the `Sample` model:

1. In the `Modalities` class, the value `MULTIPLEXED` is added as one of the modalities and to a dictionary named `NAME_MAPPING ` ([L15](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L15)):

```py
class Modalities:
      ...
      MULTIPLEXED = "MULTIPLEXED" 
      ...
      NAME_MAPPING = {
         ...
         MULTIPLEXED: "Multiplexed", 
         ...
     }
```

2. In the static method `get_output_metadata_file_path(scpca_sample_id, modality)`, it's used as a key (`MULTIPLEXED`) in the returned dictionary ([100](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L100)):
```py
 def get_output_metadata_file_path(scpca_sample_id, modality):
        return {
            Sample.Modalities.MULTIPLEXED: common.OUTPUT_DATA_PATH
            / f"{scpca_sample_id}_multiplexed_metadata.tsv",
```

And the static method above is called in the `output_multiplexed_metadata_file_path(self)` to get the path of the multiplexed metadata file [142](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L142):

```py
def output_multiplexed_metadata_file_path(self):
 return Sample.get_output_metadata_file_path(self.scpca_id, Sample.Modalities.MULTIPLEXED) # Clean up
```

At line [113](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L113), the `has_multiplexed_data` property should be removed from the `modalities`:
```py
  def modalities(self):
        attr_name_modality_mapping = {
            "has_bulk_rna_seq": Sample.Modalities.BULK_RNA_SEQ,
            "has_cite_seq_data": Sample.Modalities.CITE_SEQ,
            "has_multiplexed_data": Sample.Modalities.MULTIPLEXED, # Clean up
            "has_spatial_data": Sample.Modalities.SPATIAL,
```



