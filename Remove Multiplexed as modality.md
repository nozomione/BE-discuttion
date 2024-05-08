
Merged PR: [Multiplexed is now SingleCell modality](https://github.com/AlexsLemonade/scpca-portal/pull/672)

Epic: [Remove Sample.Modalities.MULTIPLEXED](https://github.com/AlexsLemonade/scpca-portal/issues/695) 


## `multiplexed` in the `load-data` script

At line [154](https://github.com/AlexsLemonade/scpca-portal/blob/dev/api/scpca_portal/management/commands/load_data.py#L154), the value of `has_multiplex` in `project_metadata.csv` is assigned to the `project.has_multiplexed_data` field:
```py
def process_project_data(self, data, sample_id, **kwargs):

  ...

  self.project.has_multiplexed_data = utils.boolean_from_string(data.get("has_multiplex", False)
```

<details>
<summary>The <code>process_project_data</code> method and the <code>load-data</code> command</summary><br/>
  
The `process_project_data` method is called in the `load_data` method at line [251](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/management/commands/load_data.py#L251): 

```py
def load_data(
 self,
 allowed_submitters: set[str] = None,
 input_bucket_name: str = "scpca-portal-inputs",
 **kwargs,
):
  """Loads data from S3. Creates projects and loads data for them."""

  with open(Project.get_input_project_metadata_file_path()) as project_csv:
      project_list = list(csv.DictReader(project_csv)) # project_metadata.csv

  for project_data in project_list:
    self.process_project_data(project_data, sample_id, **kwargs)
```

The `load-data` command is defined in `scpca-portal/bin/sportal` at line [34](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/bin/sportal#L34):

```py
 "load-data": run_api.format("./manage.py load_data {}"),
```
<hr/>
</details>

#### Q1: The difference between the `CommonDataAttribute` and `Sample` models's `has_multiplexed_data` field
Both [`CommonDataAttributes`](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/base.py#L12) and [`Sample`](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L34) models define the `has_multiplexed_data` field. What differentiates them and the reasons behind it. 

```py
has_multiplexed_data = models.BooleanField(default=False)
```

#### Q2: Where does the value of `multiplexed_with` come from?
At line [40](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L40) in the `Sample` model, the `multiplexed_with` field is defined: 

```py
multiplexed_with = ArrayField(models.TextField(), default=list)
```

And at line [59](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L59), its value is then assigned to the `has_multiplexed_data` field:

```py
 @classmethod
    def get_from_dict(cls, data, project):
        """Prepares ready for saving sample object."""

        has_multiplexed_data = bool(data.get("multiplexed_with"))
```

Where dose the value of `multiplexed_with` come from?

## Identify `Sample.Modalities.MULTIPLEXED` in the codebase

### The`Project` model

At line [288](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/project.py#L228), the modality constant should be removed:

```py
def combine_multiplexed_metadata(
        self,
        samples_metadata: List[Dict],
        multiplexed_libraries_metadata: List[Dict],
        combined_single_cell_metadata: List[Dict],
        sample_id: str,
    ):
 
       ...

        modality = Sample.Modalities.MULTIPLEXED ## Needs to be removed
```
#### Q1: The [`combine_multiplexed_metadata`](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/project.py#L211) method 


#### Q2: metadata json files

### The `Sample` model

At line [18](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L18) and [25](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L25), `Multiplexed` needs to be removed from the modality:

```py
class Modalities:
        BULK_RNA_SEQ = "BULK_RNA_SEQ"
        CITE_SEQ = "CITE_SEQ"
        MULTIPLEXED = "MULTIPLEXED" # Clean up
        SINGLE_CELL = "SINGLE_CELL"
        SPATIAL = "SPATIAL"

        NAME_MAPPING = {
            BULK_RNA_SEQ: "Bulk RNA-seq",
            CITE_SEQ: "CITE-seq",
            MULTIPLEXED: "Multiplexed", # Clean up
            SPATIAL: "Spatial Data",
        }
```

At line [100-101](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L100-L101), in the static method `get_output_metadata_file_path(scpca_sample_id, modality)`, a check for `Multiplexed` should be revised:
```py
 def get_output_metadata_file_path(scpca_sample_id, modality):
        return {
            Sample.Modalities.MULTIPLEXED: common.OUTPUT_DATA_PATH
            / f"{scpca_sample_id}_multiplexed_metadata.tsv",
```

This method is called at line [142](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L142):

```py
def output_multiplexed_metadata_file_path(self):
 return Sample.get_output_metadata_file_path(self.scpca_id, Sample.Modalities.MULTIPLEXED) # Clean up
```

At line [113](https://github.com/AlexsLemonade/scpca-portal/blob/2d93c9550c4fd442ad85a8568215c1c116d31146/api/scpca_portal/models/sample.py#L113), the "has_multiplexed_data" value should be removed:
```py
  def modalities(self):
        attr_name_modality_mapping = {
            "has_bulk_rna_seq": Sample.Modalities.BULK_RNA_SEQ,
            "has_cite_seq_data": Sample.Modalities.CITE_SEQ,
            "has_multiplexed_data": Sample.Modalities.MULTIPLEXED, # Clean up
            "has_spatial_data": Sample.Modalities.SPATIAL,
```



