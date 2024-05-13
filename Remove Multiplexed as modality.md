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
- Look up: List Comprehension w/ if
- Look up: Class methods
- Take what is in github markdown file and place in this document
- Create a space under each question for an answer
- From david: Prepare intro to loading of multiplexed samples
- Look at where / how we use `Sample.Model.MULTIPLEXED` this might be covered in your original notes

## 05/15/2024
