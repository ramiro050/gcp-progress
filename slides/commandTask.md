<!-- .slide: data-transition="none" -->

## CommandTask

Implemented several standard `BicepViewer` commands:

- `mce_start_acq`
- `mce_stop_acq`
- `mce_tes_bias`
- `mce_get_runfile`
- `mceGeneric`

***

<!-- .slide: data-transition="none" -->

## CommandTask

Implemented several standard `BicepViewer` commands:

- `mce_start_acq` <!-- .element: style="color:rgba(255,255,255,0.3);"-->
- `mce_stop_acq` <!-- .element: style="color:rgba(255,255,255,0.3);"-->
- `mce_tes_bias`
- `mce_get_runfile` <!-- .element: style="color:rgba(255,255,255,0.3);"-->
- `mceGeneric` <!-- .element: style="color:rgba(255,255,255,0.3);"-->

***

[`mce_tes_bias` documentation](http://bicep.rc.fas.harvard.edu/bicep_array/computers/gcp_usage/CommandIndex.html)

***

<!-- .slide: data-auto-animate -->

#### `bin/control/specificscript.cc`

```cpp [|9-10]
Script* add_SpecificScriptCommmands(ControlProg *cp,
                                    Script* sc,
                                    int batch,
                                    HashTable *signals)
{
  // some code

  if(!add_BuiltinCommand(sc,
                         "mce_tes_bias([Integer dac, listof Integer dacs, \
                                        Generic col, Mces mce])",
                         sc_mceTesBias_cmd))
    return del_Script(sc);

  // more code
}
```

***

<!-- .slide: data-auto-animate -->

#### `bin/control/specificscript.cc`

```cpp [|74-88|90-109|110-113]
static CMD_FN(sc_mceTesBias_cmd)
{
  Variable* var_dac;
  Variable* var_col;
  Variable* var_dacs;
  Variable* var_mces;

  ControlProg *cp = (ControlProg* )sc->project; /* The control-program
                                                   resource object */
  RtcNetCmd rtc;             // The network object to be sent to
  // the real-time controller task
  // Do we have a controller to send the command to?
  if(rtc_offline(sc, "mce_tes_bias"))
    return 1;

  if(get_Arguments(args, &var_dac, &var_dacs, &var_col, &var_mces, NULL))
    return 1;

  // Default to all mces.
  rtc.receivers = OPTION_HAS_VALUE(var_mces) ?
    SET_VARIABLE(var_mces)->set : cp_MceSet(cp)->getId();

  if (!OPTION_HAS_VALUE(var_col))
    {
      rtc.cmd.mce_tes_bias.col = ~0;
    }
  else
    {
      // Don't allow pathological things
      if (!OPTION_HAS_VALUE(var_mces)) {
        ReportMessage("If col is specified, mces must be specified as well");
        return 1;
      }

      // Make a mask to represent all BA MCEs
      unsigned ba_mask = 0;
      for (int i = 0; i<N_BA_MCE; i++)
        ba_mask |= 1<<i;
      // Make a mask to represent all Keck MCEs
      unsigned keck_mask = 0;
      for (int i = N_BA_MCE; i<(N_BA_MCE+N_KECK_MCE); i++)
        keck_mask |= 1<<i;
      // Make a mask to represent all BA SMuRFs
      unsigned smurf_mask = 0;
      for (int i = (N_BA_MCE+N_KECK_MCE); i<(N_BA_MCE+N_KECK_MCE+N_BA_SMURF); i++)
        smurf_mask |= 1<<i;
      // Are we setting different MCE types?
      if (((rtc.receivers & ba_mask) && (rtc.receivers & keck_mask)) ||
          ((rtc.receivers & ba_mask) && (rtc.receivers & smurf_mask)) ||
          ((rtc.receivers & keck_mask) && (rtc.receivers & smurf_mask)))
        {
          ReportMessage("When col is specified, mces must be either BA or Keck MCEs, or SMuRFs only");
          return 1;
        }

      if (rtc.receivers & ba_mask) {
        if (parse_mask_str(&(rtc.cmd.mce_tes_bias.col),
                           STRING_VARIABLE(var_col)->string,
                           COLUMNS_PER_BA_MCE-1))
          return 1;
      } else if (rtc.receivers & keck_mask) {
        if (parse_mask_str(&(rtc.cmd.mce_tes_bias.col),
                           STRING_VARIABLE(var_col)->string,
                           COLUMNS_PER_KECK_MCE-1))
          return 1;
      } else {
        if (parse_mask_str(&(rtc.cmd.mce_tes_bias.col),
                           STRING_VARIABLE(var_col)->string,
                           SMURF_BIAS_GROUPS-1))
          return 1;
      }
    }

  // create the command that will be sent to the real-time controller
  // Here we check that the arguments are correctly specified
  if (OPTION_HAS_VALUE(var_dac)) {
    if (OPTION_HAS_VALUE(var_dacs)) {
      ReportMessage("If dac is specified, dacs must not be specified.");
      return 1;
    }

    rtc.cmd.mce_tes_bias.dac  = (int)(INT_VARIABLE(var_dac)->i);
    rtc.cmd.mce_tes_bias.using_dacs = false;
  } else if (OPTION_HAS_VALUE(var_dacs)) {
    if (OPTION_HAS_VALUE(var_col)) {
      ReportMessage("If dacs is specified, col must not be specified.");
      return 1;
    }

    size_t curr_index = 0;
    for(ListNode *node = LIST_VARIABLE(var_dacs)->list->head;
        node; node = node->next) {

      if (curr_index >= SMURF_BIAS_GROUPS) {
        ReportMessage("Array of dacs must be of length " << SMURF_BIAS_GROUPS);
        return 1;
      }

      int bias = INT_VARIABLE(node->data)->i;
      rtc.cmd.mce_tes_bias.dacs[curr_index] = bias;
      curr_index++;
    }

    if (curr_index < SMURF_BIAS_GROUPS) {
      ReportMessage("Array of dacs must be of length " << SMURF_BIAS_GROUPS);
      return 1;
    }

    rtc.cmd.mce_tes_bias.using_dacs = true;
  } else {
    ReportMessage("dac or dacs must be specified.");
    return 1;
  }

  // Send the command to the real-time controller.
  if(queue_rtc_command(cp, &rtc, NET_MCE_TES_BIAS_CMD))
    return 1;

  COUT("Sending MCE_TES_BIAS command.");

  return 0;
}

```

***

#### `bin/smurfd/include/smurfd/CommandMsg.h`

```cpp [5-10]
class CommandMsg : public gcp::util::GenericMessage {
public:
  // some code

  struct TesBiasMembers {
    int dac;
    int dacs[SMURF_BIAS_GROUPS];
    bool using_dacs;
    NetMask col_mask;
  };

  // more code
};


```

***

#### `bin/smurfd/include/smurfd/CommandTask.cc.in`

```cpp [|14-20|21-29]
/**.......................................................................
 * Bias TESs
 */
void CommandTask::runTesBiasCommand(CommandMsg* msg)
{
  //DEBUGGING
  COUT("smurfd::CommandTask::runTesBiasCommand()");

  CommandMsg::TesBiasMembers t = msg->body.tes_bias_members;
  unsigned int slot = parent_->getSlot();
  ostringstream oss;
  oss << "@CMAKE_INSTALL_PREFIX@/scripts/smurfd/smurf_cmd.py -v \"";

  if (t.using_dacs) {
    oss << "--tes-bias --bias-voltage-array \\\"";

    for (int col = 0; col < SMURF_BIAS_GROUPS; col++) {
      if (col != 0) oss << " ";
      oss << t.dacs[col];
    }
  } else {
    oss << "--tes-bias --bias-voltage " << t.dac << " --bias-group \\\"";

    bool first_col_added = false;
    for (int on_col = 0; on_col < SMURF_BIAS_GROUPS; on_col++) {
      if ((t.col_mask >> on_col) & 1) {
        if (first_col_added) oss << " ";
        oss << on_col;
        first_col_added = true;
      }
    }
  }

  oss << "\\\"\" " << slot << " &";

  COUT("Running: " << oss.str());
  // run the shell command
  system(oss.str().c_str());
}
```

***

## demo

```shell []
bicepViewer> mce_tes_bias
bicepViewer> mce_tes_bias dac=10, dacs={1,2,3}
bicepViewer> mce_tes_bias dacs={1,2,3}, col=3, mce=mce1
bicepViewer> mce_tes_bias dacs={1,2,3}
bicepViewer> mce_tes_bias dacs={1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17}
bicepViewer> mce_tes_bias dacs={1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16}
bicepViewer> mce_tes_bias dac=10, col=2+5, mce=mce1
```

notes:
1. Error: dac or dacs
2. Error: If dac, then no dacs
3. Error: If dacs, no col (mce must be there for col)
4. Error: too small
5. Error: too large
6. Correct
7. Correct
