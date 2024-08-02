************************************************************************
* Program Name      : YGSTN_INWARD_MAIN_V1                             *
* Description       : EY INWARD SUPPLY EXTRACTOR PROGRAM                  *
* Create Date       :                                                  *
* FO Owner          :                                                  *
* Tech Owner        :                                                  *
************************************************************************
*                        Change Log                                    *
************************************************************************
*  REQ#         DATE         WHO            CHANGE_ID        DESCR     *
*-----------------------------------------------------------------------
*  001        21-09-20    Javed Rana         JVR001      Include reverse *
*                                                        documents &
*                                                        supplytype =
*                                                        'CAN' for MR8M
*                                                        and FB08 doc
************************************************************************`
REPORT ygstn_inward_main_v1.

INCLUDE ygstn_inward_main_v1_top.

INCLUDE ygstn_inward_main_v1_sel.

INCLUDE ygstn_inward_main_v1_inward.

INCLUDE ygstn_inward_main_v1_adv_rcm.

INCLUDE ygstn_inward_main_v1_adv_adj.

INCLUDE ygstn_inward_main_v1_itc_rev.

INCLUDE ygstn_inward_main_v1_tds.

INCLUDE ygstn_inward_main_v1_sub.

START-OF-SELECTION.
  """""""""""""""""""""""""Setting Flag abd Global variables"""""""""""""""""""""""""""""""
  CLEAR: gv_get_live_data, gv_get_staging_data.
  IF r_file = 'X'.
    IF r_live = 'X'.
      gv_get_live_data = 'X'.
    ELSEIF r_stage = 'X'.
      gv_get_staging_data = 'X'.
    ENDIF.
  ELSEIF r_backup = 'X'.
    gv_get_live_data = 'X'.
  ELSEIF r_test = 'X'.
    IF r_live = 'X'.
      gv_get_live_data = 'X'.
    ELSEIF r_stage = 'X'.
      gv_get_staging_data = 'X'.
    ENDIF.
  ENDIF.
  CLEAR gr_budat.
  IF r_date = 'X'.
    gr_budat[] = s_date[].
  ELSEIF r_month = 'X'.
    PERFORM populate_date_from_month.
  ENDIF.
  """"""""""""""""""""""""""""""Inward supply""""""""""""""""""""""""""""""""

  IF ( ( r_file = 'X' OR r_backup = 'X' ) AND  c_imain = 'X' ) OR ( r_test = 'X' AND r_imain = 'X' ).
    IF gv_get_live_data = 'X'.
      PERFORM generate_main_inward_from_live.
    ELSEIF gv_get_staging_data = 'X'.
      PERFORM get_main_inward_staging.
    ENDIF.
    IF r_file ='X'.
      PERFORM create_return_log USING gc_code_inward.
    ENDIF.
    IF r_backup <> 'X'.

      ASSIGN gt_final_inward TO <fs_table>.
      PERFORM display_alv_or_create_file TABLES <fs_table>
      USING gc_tablename_inward
            gc_structure_inward
            gc_code_inward
            gc_folder_inward.
    ELSE.
      PERFORM create_backup_main_inward.
    ENDIF.
  ENDIF.

  """"""""""""""""""""""""""""""""Advance RCM""""""""""""""""""""""""""""""""""""""
  IF ( ( r_file = 'X' OR r_backup = 'X' ) AND  c_adrcm = 'X' ) OR ( r_test = 'X' AND r_adrcm = 'X' ).
    IF gv_get_live_data = 'X'.
      PERFORM generate_advance_rcm_from_live.
    ELSEIF gv_get_staging_data = 'X'.
      PERFORM get_advance_rcm_staging.
    ENDIF.
    IF r_file ='X'.
      PERFORM create_return_log USING gc_code_advpaidrcm.
    ENDIF.
    IF r_backup <> 'X'.

      ASSIGN gt_final_advpaidrcm TO <fs_table>.
      PERFORM display_alv_or_create_file TABLES <fs_table>
      USING gc_tablename_advpaidrcm
            gc_structure_advpaidrcm
            gc_code_advpaidrcm
            gc_folder_advpaidrcm.
    ELSE.
      PERFORM create_backup_advance_rcm.
    ENDIF.
  ENDIF.

  """"""""""""""""""""""""""""""""""Advance adjusted"""""""""""""""""""""""""""""
  IF ( ( r_file = 'X' OR r_backup = 'X' ) AND  c_ajrcm = 'X' ) OR ( r_test = 'X' AND r_ajrcm = 'X' ).
    IF gv_get_live_data = 'X'.
      PERFORM generate_adv_adj_rcm_from_live.
    ELSEIF gv_get_staging_data = 'X'.
      PERFORM get_adv_adj_rcm_staging.
    ENDIF.
    IF r_file ='X'.
      PERFORM create_return_log USING gc_code_advadjrcm.
    ENDIF.
    IF r_backup <> 'X' AND gt_final_inward IS NOT INITIAL.
*  ******Code Added by Javed
      gt_final_inward_inv = gt_final_inward = gt_final_inward.
      DELETE gt_final_inward_inv WHERE documenttype <> 'INV'.
      DELETE gt_final_inward_in WHERE documenttype = 'INV'.
      IF gt_final_inward_inv IS NOT INITIAL.
        ASSIGN gt_final_inward_inv TO <fs_table>.
        PERFORM display_alv_or_create_file TABLES <fs_table>
        USING gc_tablename_inward
              gc_structure_inward
              gc_code_inward
              gc_folder_inward.
        IF <fs_table> IS ASSIGNED.
          UNASSIGN <fs_table>.
        ENDIF.
      ENDIF.
      IF gt_final_inward_in IS NOT INITIAL.
        ASSIGN gt_final_inward_in TO <fs_table>.
        PERFORM display_alv_or_create_file TABLES <fs_table>
        USING gc_tablename_inward
              gc_structure_inward
              gc_code_inward
              gc_folder_inward.
        IF <fs_table> IS ASSIGNED.
          UNASSIGN <fs_table>.
        ENDIF.
      ENDIF.
*  ******Code Added by Javed
*    IF r_backup <> 'X'.
*      ASSIGN gt_final_advadjrcm TO <fs_table>.
*      PERFORM display_alv_or_create_file TABLES <fs_table>
*      USING gc_tablename_advadjrcm
*            gc_structure_advadjrcm
*            gc_code_advadjrcm
*            gc_folder_advadjrcm.
    ELSE.
      PERFORM create_backup_adv_adj_rcm.
    ENDIF.
  ENDIF.

  """""""""""""""""""""""""""""""""""""""""""ITC reversal""""""""""""""""""""""""""""""

  IF ( ( r_file = 'X' OR r_backup = 'X' ) AND  c_itcrev = 'X' ) OR ( r_test = 'X' AND r_itcrev = 'X' ).
    IF gv_get_live_data = 'X'.
      PERFORM generate_itc_rev_from_live.
    ELSEIF gv_get_staging_data = 'X'.
      PERFORM get_itc_rev_staging.
    ENDIF.
    IF r_file ='X'.
      PERFORM create_return_log USING gc_code_itcreversal.
    ENDIF.
    IF r_backup <> 'X'.
      ASSIGN gt_final_itcreversal TO <fs_table>.
      PERFORM display_alv_or_create_file TABLES <fs_table>
      USING gc_tablename_itcreversal
            gc_structure_itcreversal
            gc_code_itcreversal
            gc_folder_itcreversal.
    ELSE.
      PERFORM create_backup_itc_reversal.
    ENDIF.
  ENDIF.

  """""""""""""""""""""""""""""""""""""ISD MAP"""""""""""""""""""""""""""""""""""""""
  IF ( ( r_file = 'X' OR r_backup = 'X' ) AND  c_isdmap = 'X' ) OR ( r_test = 'X' AND r_isdmap = 'X' ).

    """"""""""""""""""""Functionality to be added""""""""""""""""""""""""""
  ENDIF.

  """""""""""""""""""""""""""""""""""""TDS"""""""""""""""""""""""""""""""""""""""""""
  IF ( ( r_file = 'X' OR r_backup = 'X' ) AND  c_tds = 'X' ) OR ( r_test = 'X' AND r_tds = 'X' ).
    IF gv_get_live_data = 'X'.
      PERFORM generate_tds_from_live.
    ELSEIF gv_get_staging_data = 'X'.
      PERFORM get_tds_staging.
    ENDIF.
    IF r_file ='X'.
      PERFORM create_return_log USING gc_code_tds.
    ENDIF.
    IF r_backup <> 'X'.
      ASSIGN gt_final_tds TO <fs_table>.
      PERFORM display_alv_or_create_file TABLES <fs_table>
      USING gc_tablename_tds
            gc_structure_tds
            gc_code_tds
            gc_folder_tds.
    ELSE.
      PERFORM create_backup_tds_rev.
    ENDIF.
  ENDIF.
