# Review of Petronas Form RAP Implementation

## Summary
The provided implementation covers a substantial amount of RAP functionality, but there are several issues that would prevent the solution from compiling or behaving correctly in a managed RAP scenario. The most critical points are highlighted below with recommendations.

## Findings

### 1. Invalid `SELECT SINGLE ... FOR ALL ENTRIES`
In `ValidateCompanyCode` a `SELECT SINGLE` is combined with `FOR ALL ENTRIES`:

```abap
SELECT SINGLE @abap_true
  FROM t001
  FOR ALL ENTRIES IN @lt_forms
  WHERE bukrs = @lt_forms-Zcompcode
  INTO @DATA(lv_exists).
```

`SELECT SINGLE` cannot be used together with `FOR ALL ENTRIES`; the statement will not compile. Even if it did, the logic would only check one company code. Use a set-based `SELECT ... FOR ALL ENTRIES` (without `SINGLE`) and validate each key individually.

### 2. Activity log performs an explicit `COMMIT WORK`
`write_activity_log` inserts into `zmdf_log` and then issues `COMMIT WORK`. Behavior implementations must not manage their own LUW. Let the RAP framework commit by removing the `COMMIT WORK`; otherwise draft/transaction handling will break.

### 3. Totals accumulate across forms
In `CalculateTotals` the helper variables declared outside the loop keep their values between forms:

```abap
DATA: lv_tot_weight TYPE ztotweightton,
      lv_tot_cost   TYPE ztotnetwr.

LOOP AT lt_forms INTO DATA(ls_form).
  ...
  LOOP AT lt_mdfs INTO DATA(ls_mdf).
    lv_tot_weight = lv_tot_weight + ls_mdf-ZmdfTotWeightTon.
    lv_tot_cost   = lv_tot_cost   + ls_mdf-ZmdfTotPurchCost.
  ENDLOOP.

  MODIFY ENTITIES ...
ENDLOOP.
```

The accumulators must be cleared per form; otherwise the totals keep growing. Add `CLEAR: lv_tot_weight, lv_tot_cost.` at the start of each outer loop iteration (or declare them inside the loop).

### 4. Mass-enabled action only handles the first key
`CreateFromSelection` reads only `keys[ 1 ]`. If the action is triggered for multiple instances, the additional keys are ignored. Either declare the action as not mass-enabled or loop over `keys` and handle each one.

### 5. Missing `FOR ALL ENTRIES` guard
Methods such as `FetchMdfData` and `ValidateMdfStatus` run `SELECT ... FOR ALL ENTRIES IN @lt_maps` without checking whether `lt_maps` is initial. Add `IF lt_maps IS INITIAL. RETURN. ENDIF.` guards to prevent unintended full-table selects.

### 6. Late-numbering lookup on draft table
`adjust_numbers` reads from the draft table `zscmimd007_head` using `%pid`. In managed RAP, `%pid` does not correspond to persisted draft entries once the instance is activated. Consider reading the current instances via the `create`/`mapped` structures provided by the saver, or switch to a released number range object.

## Recommendations
Address the issues above before deploying the solution. After the fixes, re-run the RAP contract checks and functional tests to ensure that draft handling, totals calculation, and number assignment behave as expected.
