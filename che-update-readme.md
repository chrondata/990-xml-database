# CHE Update Readme Oct. 8, 2020

CHE uses MySQL-dialect database servers rather than PostgreSQL, so there are additional steps that need to be taken when updating the CHE fork of this repo to the upstream version.
*Before you start, be sure to take a snapshot of the 990-xml-database volume in AWS so that you have a restore point should this go horribly wrong.*

## Merge forked feature branches into branch `master`
This will minimize conflicts between our working code and the upstream version. If there is no feature branch, move on to the next step.

I did this merge on the GitHub site, and it merged without conflict.

## Merge upstream master branch into forked master branch
There will likely be conflicts to resolve, likely in `return/models.py`. Our forked version of this repo namely refers to two fields differently from the upstream repo. This is a necessity because the field names from the upstream repo are identical when case is ignored, and MySQL ignores casing in field names, while Postgres does not.

I did this merge using the GitHub Desktop app, which enables me to edit conflicts in my own text editor.

When resolving these conflicts, take note of all field names that end up varying from the field names in the upstream version.

### pf_part_v
Upstream master field name: `AppldTESTxAmt`
Forked master field name (CHE): `AmtCrdtNxtYr`

Check the comments to verify that `AppldTESTxAmt` refers to:
>Line number: Part VI Line 6c  Description: Tax Paid with Extension  most recent xpath: /IRS990PF/ExciseTaxBasedOnInvstIncmGrp/ExtsnRequestIncomeTaxPaidAmt

Do not confuse this field with `AppldTEsTxAmt`:
>Line number: Part VI Line 6c  Description: Tax Paid with Extension  most recent xpath: /IRS990PF/ExciseTaxBasedOnInvstIncmGrp/ExtsnRequestIncomeTaxPaidAmt

### SkdaSpprtdOrgInfmtn
Upstream master field name: `EIN`
Forked master field name (CHE): `SpprtdOrgEIN`

Check the comments to verify that `EIN` refers to:
>Line number:  Part I Line 12g Column (ii)  Description:  EIN of supported organization  most recent xpath: /IRS990ScheduleA/SupportedOrgInformationGrp/EIN

Do not confuse this field with `ein`, which is the `ein` field for the organization filing this return.

Continue resolving all other merge conflicts presented. Accept as many of the `upstream` merge conflicts as possible only after verifying which `origin` fields are being replaced (if they are at all) and editing as such to prevent any local conflicts for fields that refer to the same value. Check the comments on each to verify what part of the form these fields actually refer.

## Pull merged code onto server
```sh
git fetch
git pull upstream master
```

## Update IRSx
```sh
pip install --upgrade irsx
```
This is the module that actually processes XMLs into return data. It is updated separately from the core repo.

## Make variables.csv edits in IRSx
Identify where your IRSx settings file is stored by entering this into the Python shell while the virtual environment is open:
```python
from irsx.settings import IRSX_SETTINGS_LOCATION
IRSX_SETTINGS_LOCATION
>>> '/long/path/to/lib/python3.6/site-packages/irsx/settings.py'
```

Open the settings file in your text editor of choice and identify to where `METADATA_DIRECTORY` leads. `cd` into `METADATA_DIRECTORY`.

Identify the field names that we altered from their upstream counterparts. Take special note of the forms and sections (possibly schedules as well) from which these fields came because you need this information to verify the field names we'll be altering to match.

If you are unfamiliar with `variables.csv` in this `METADATA_DIRECTORY`, I recommend opening it first in a spreadsheet to familiarize yourself with the format. Once familiarized, use your text editor (or spreadsheet editor) of choice to search for the **upstream field names** that we altered. Identify which field represents the actual field name / variable name, and edit that field name to match the forked counterpart to the upstream field name. Match XPaths between `models` and `variables` where possible. For example:

This row:

```csv
IRS990ScheduleA,skeda_part_i,True,SkdASpprtdOrgInfrmtn,315,EIN,/IRS990ScheduleA/SupportedOrgInformationGrp/EIN,EINType,String(length=9),Part I Line 12g Table; Part I Line 12g Column (ii),Information about the supported organizations; EIN of supported organization,2013,
```

Becomes this row:

```csv
IRS990ScheduleA,skeda_part_i,True,SkdASpprtdOrgInfrmtn,315,SpprtdOrgEIN,/IRS990ScheduleA/SupportedOrgInformationGrp/EIN,EINType,String(length=9),Part I Line 12g Table; Part I Line 12g Column (ii),Information about the supported organizations; EIN of supported organization,2013,

```

This row:
```csv
IRS990PF,pf_part_vi,False,pf_part_vi,788,AppldTESTxAmt,/IRS990PF/ExciseTaxBasedOnInvstIncmGrp/AppliedToESTaxAmt,USAmountNNType,BigInteger,Part VI Line 11,Amount Credited to Next Year,2013,
```

Becomes this row:
```csv
IRS990PF,pf_part_vi,False,pf_part_vi,788,AmtCrdtNxtYr,/IRS990PF/ExciseTaxBasedOnInvstIncmGrp/AppliedToESTaxAmt,USAmountNNType,BigInteger,Part VI Line 11,Amount Credited to Next Year,2013,
```

Check, double-check, and triple-check that these field names are identical to the field names in `return/models.py`.

Additionally, I would recommend against pasting the changes here into `variables.csv` as other portions of the row may have changed. Replace the variable name only with our forked variable name.

If you do not do this step properly, the 990 XML parser will throw a `KeyError` on whatever fields were labeled improperly when processing returns into the database.


## Migrate models
Navigate to `manage.py` and make your migrations:
```sh
python manage.py makemigrations
```

Follow the interactive shell to confirm/reject changes. It should be safe to accept changes, but **please** verify that Django is picking up the correct name changes before blindly accepting.

If all goes well, the parser should now be updated and new returns can be processed into the database.
