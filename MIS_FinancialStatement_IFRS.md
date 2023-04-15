# FINANCIAL REPORTS FOR BOM LEVEL
## 1. ENR MOVEMENT

       create or replace view v_fin_sales_enr_movement as
       with dtl_enr as
       (
       select 'ENR' items
              , CASE WHEN PRODUCT ='CD' THEN 'CDL'
                       WHEN PRODUCT ='TU' THEN 'PLTU'
                         WHEN PRODUCT ='XS' THEN 'PLXS'
                           ELSE TO_CHAR(PRODUCT) END AS PRODUCT
              , TO_CHAR(MONTHID) monthid
              , sum(amt_)/1000 value
       from risk_nhannt.tv_fin_investordb_enr_prod_sas t
       where ifrs_vas = 'VAS'
       group by CASE WHEN PRODUCT ='CD' THEN 'CDL'
                       WHEN PRODUCT ='TU' THEN 'PLTU'
                         WHEN PRODUCT ='XS' THEN 'PLXS'
                           ELSE TO_CHAR(PRODUCT) END
                , TO_CHAR(MONTHID)
       )
       , dtl as
       (
       select TO_CHAR(ITEMS) ITEMS, TO_CHAR(PRODUCT) PRODUCT, MONTHID, VALUE --month
       from IMPT_FIN_SALES t
       union all
       select TO_CHAR(ITEMS) ITEMS, TO_CHAR(PRODUCT) --quarter
              , case when substr(monthid,5,2) in ('01','02','03') then substr(monthid,1,4) || 'Q1'
                       when substr(monthid,5,2) in ('04','05','06') then substr(monthid,1,4) || 'Q2'
                         when substr(monthid,5,2) in ('07','08','09') then substr(monthid,1,4) || 'Q3'
                           ELSE substr(monthid,1,4) || 'Q4'
                             end as monthid
              , sum(value) value
       from IMPT_FIN_SALES t
       group by TO_CHAR(ITEMS), TO_CHAR(PRODUCT)
                , case when substr(monthid,5,2) in ('01','02','03') then substr(monthid,1,4) || 'Q1'
                         when substr(monthid,5,2) in ('04','05','06') then substr(monthid,1,4) || 'Q2'
                           when substr(monthid,5,2) in ('07','08','09') then substr(monthid,1,4) || 'Q3'
                             ELSE substr(monthid,1,4) || 'Q4'
                               end
       union all
       select TO_CHAR(ITEMS) ITEMS, TO_CHAR(PRODUCT) --year
              , 'FY' || substr(monthid,1,4) as monthid
              , sum(value) value
       from IMPT_FIN_SALES t
       group by TO_CHAR(ITEMS), TO_CHAR(PRODUCT)
                , 'FY' || substr(monthid,1,4)
       )
       , SUMM as
       (
       select TO_CHAR(ITEMS) ITEMS, TO_CHAR(PRODUCT) PRODUCT, TO_CHAR(MONTHID) MONTHID, VALUE  from dtl
       union all
       select TO_CHAR(ITEMS) ITEMS, TO_CHAR(PRODUCT) PRODUCT, TO_CHAR(MONTHID) MONTHID, VALUE from dtl_enr
       where monthid not in ('FY2016','FY2017')
       )
       select t.items
              , case when product = 'DCARD' then 'CDL'
                       WHEN product = 'PL_CASH24' then 'PLNTB'
                         ELSE PRODUCT
                           END AS PRODUCT
              , monthid
              , sum(value) value
              --, t.*
              , case when monthid like '%Q%' then '2.QUARTERLY'
                       when monthid like '%FY%' then '4.YEARLY'
                         else '1.MONTHLY' end as time_type
       from summ t
       GROUP BY t.items
                , case when product = 'DCARD' then 'CDL'
                         WHEN product = 'PL_CASH24' then 'PLNTB'
                           ELSE PRODUCT
                             END
                , monthid
                , case when monthid like '%Q%' then '2.QUARTERLY'
                         when monthid like '%FY%' then '4.YEARLY'
                           else '1.MONTHLY' end;

## 2. P&L at BOM level

       CREATE OR REPLACE VIEW V_FIN_PL_FC_EXT_MANAGEMENT_3 AS
       with dtl as
       (
       select *
       from ( select monthid
                     , time_type
                     , amt_type
                     , description_eng
                     , amt
              from V_FIN_PL_FC_EXT_MANAGEMENT_3_DTL t
              where amt_type in ('MGMT_AMT_FINAL','AUDIT_AMT')
             )
       pivot ( sum(amt) for (description_eng)
               in ('TOI' TOI
                   ,'Fees and commission expenses' Service_expenses
                   ,'Net interest and similar income (NII)' NII
                   ,'TOTAL OPERATING EXPENSES (OPEX)' OPEX
                   ,'Credit loss expense' PROVISION
                   ,'Net other operating income' OTHER_INCOME
                   ,'NCL' NCL
                   ,'RAR' RAR )
             )
       )
       , finally as
       (
       select t.*, ANR, ENR
       from dtl t
       left join ( select T.monthid
                          , CASE WHEN T.time_type ='MONTHLY' THEN '1.MONTHLY'
                                   WHEN T.time_type ='QUARTERLY' THEN '2.QUARTERLY'
                                     WHEN T.time_type ='HALF_YEAR' THEN '3.HALF_YEAR'
                                       WHEN T.time_type ='YTD' THEN '4.YEARLY'
                                         end as time_type
                          , T.amt_ ENR
                          , K.AMT_ ANR
                   from (SELECT * FROM V_FIN_ENR_NPL where items = 'TOTAL_ENR(1-5)' and product_org ='ALL' AND IFRS_VAS ='VAS') T
                   LEFT JOIN (SELECT * FROM V_FIN_ENR_NPL where items = 'TOTAL_ANR' and product_org ='ALL' AND IFRS_VAS ='VAS') K on T.MONTHID=K.MONTHID AND T.TIME_TYPE=K.TIME_TYPE
                   ) k ON T.MONTHID=K.MONTHID AND T.TIME_TYPE=K.TIME_TYPE
       )
       , summ as
       (
       select monthid
              , TIME_TYPE
              , amt_type
              , enr
              , anr
              , CASE WHEN ANR = 0 THEN NULL ELSE TOI/ANR END AS TOI_ANR
              , CASE WHEN ANR = 0 THEN NULL ELSE NII/ANR END AS NIM
              , CASE WHEN ANR = 0 THEN NULL ELSE PROVISION/ANR END AS PROVISION_ANR
              , CASE WHEN TOI = 0 THEN NULL ELSE (nvl(OPEX,0)+nvl(Service_expenses,0))/TOI END AS CIR
              , CASE WHEN ANR = 0 THEN NULL ELSE NCL/ANR END AS NCL_ANR
              , CASE WHEN ANR = 0 THEN NULL ELSE RAR/ANR END AS RAR_ANR
       from finally t
       WHERE time_type = '4.YEARLY'
       UNION ALL
       select monthid
              , time_type
              , amt_type
              , enr
              , anr
              , CASE WHEN ANR = 0 THEN NULL ELSE TOI*(12/3)/ANR END AS TOI_ANR
              , CASE WHEN ANR = 0 THEN NULL ELSE NII*(12/3)/ANR END AS NIM
              , CASE WHEN ANR = 0 THEN NULL ELSE PROVISION*(12/3)/ANR END AS PROVISION_ANR
              , CASE WHEN TOI = 0 THEN NULL ELSE (nvl(OPEX,0)+nvl(Service_expenses,0))/TOI END AS CIR
              , CASE WHEN ANR = 0 THEN NULL ELSE NCL*(12/3)/ANR END AS NCL_ANR
              , CASE WHEN ANR = 0 THEN NULL ELSE RAR*(12/3)/ANR END AS RAR_ANR
       from finally t
       WHERE time_type = '2.QUARTERLY'
       UNION ALL
       select monthid
              , time_type
              , amt_type
              , enr
              , anr
              , CASE WHEN ANR = 0 THEN NULL ELSE TOI*12/ANR END AS TOI_ANR
              , CASE WHEN ANR = 0 THEN NULL ELSE NII*12/ANR END AS NIM
              , CASE WHEN ANR = 0 THEN NULL ELSE PROVISION*12/ANR END AS PROVISION_ANR
              , CASE WHEN TOI = 0 THEN NULL ELSE (nvl(OPEX,0)+nvl(Service_expenses,0))/TOI END AS CIR
              , CASE WHEN ANR = 0 THEN NULL ELSE NCL*12/ANR END AS NCL_ANR
              , CASE WHEN ANR = 0 THEN NULL ELSE RAR*12/ANR END AS RAR_ANR
       from finally t
       WHERE time_type = '1.MONTHLY'
       )
       select monthid, time_type, amt_type, items, val
       from ( select monthid, time_type, amt_type, enr, anr, toi_anr, nim, provision_anr, cir, ncl_anr, rar_anr
              from summ
              ) t
       unpivot (val for items in (enr, anr, toi_anr, nim, provision_anr, cir, ncl_anr, rar_anr))
       union all
       select t.monthid, t.time_type, t.amt_type
              , case when t.DESCRIPTION_ENG ='TOTAL OPERATING EXPENSES (OPEX)' then 'OPEX'
                       when t.DESCRIPTION_ENG ='Net other operating income' then 'Other Income'
                         when t.DESCRIPTION_ENG ='Credit loss expense' then 'Provision'
                           when t.DESCRIPTION_ENG ='PROFIT BEFORE TAX' then 'PBT'
                              when t.DESCRIPTION_ENG ='Fees and commission expenses' then 'Service expenses'
                             else t.DESCRIPTION_ENG end as DESCRIPTION_ENG
              , t.amt
       from V_FIN_PL_FC_EXT_MANAGEMENT_3_DTL t
       where t.DESCRIPTION_ENG in ('TOI','TOTAL OPERATING EXPENSES (OPEX)','Net other operating income','Credit loss expense','PROFIT BEFORE TAX','Fees and commission expenses');

## 3. SUB P&L at BOM level

       create or replace view v_fin_pl_fc_ext_management_3_dtl as
       with dtl as
       ( -- CREATE TABLE DTL AS
       select MONTHID
              , TIME_TYPE
              , AMT_TYPE
              , TRIM(DESCRIPTION_ENG) DESCRIPTION_ENG
              , fs_code
              , AMT/1000 amt
              , AMT*TYPE_/1000 AMT_MDF
              , order_
       from RISK_NHANNT.TV_FIN_INVESTORDB_PL_HL_SAS
       where order_ not in (22,5,29,30,19,20,21,9,10)
       )
       , summ as
       (
       select t.monthid, t.time_type, t.amt_type, DESCRIPTION_ENG, AMT, AMT_MDF
       from dtl T
       where DESCRIPTION_ENG <> 'TOTAL OPERATING INCOME (TOI)'
       union all
       select t.monthid, t.time_type, t.amt_type, 'TOI' DESCRIPTION_ENG
              , SUM(NVL(T.AMT,0)) amt
              , SUM(NVL(T.AMT_MDF,0)) amt_mdf
       from dtl T
       where ORDER_ IN (2 -- NII
                       ,6 -- Fee & commision income (service income)
                       ,8 -- Net FOREX
                       ,11 -- Net trading securities
                       ,12 -- Net investment securities
                       ,17 -- Income from long-term investment
                       )
       GROUP BY t.monthid, t.time_type, t.amt_type, 'TOI'
       union all
       select t.monthid, t.time_type, t.amt_type, 'NCL' DESCRIPTION_ENG
              , nvl(t.amt,0)-nvl(k.amt,0)+nvl(k1.amt,0) amt
              , nvl(t.AMT_MDF,0)-nvl(k.AMT_MDF,0)+nvl(k1.AMT_MDF,0) amt_mdf
       from (select * from dtl where DESCRIPTION_ENG = 'Credit loss expense' /*Provision*/) t
       left join (select * from dtl where DESCRIPTION_ENG = 'Recovery Income') k on t.monthid=k.monthid and t.amt_type=k.amt_type
       left join (select * from dtl where DESCRIPTION_ENG = 'Recovery Cost') k1 on t.monthid=k1.monthid and t.amt_type=k1.amt_type
       )
       select "MONTHID","TIME_TYPE","AMT_TYPE","DESCRIPTION_ENG","AMT","AMT_MDF" from summ --where monthid='202102'
       union all
       select t.monthid, t.time_type, t.amt_type, 'RAR' DESCRIPTION_ENG
              , nvl(t.amt,0)-nvl(k.amt,0) amt
              , nvl(t.amt_mdf,0)-nvl(k.AMT_MDF,0) amt_mdf
       from (select * from summ where DESCRIPTION_ENG = 'TOI') t
       left join (select * from summ where DESCRIPTION_ENG = 'NCL') k on t.monthid=k.monthid and t.amt_type=k.amt_type;

### 3.2 ENR NPL 

       create or replace view v_fin_enr_npl as
       select items
              , ifrs_vas
              , company
              , product_org
              , monthid
              , CASE WHEN ITEMS IN ('TOTAL_NPL','TOTAL_WO/ANR') THEN amt_ ELSE AMT_/1000 END AS AMT_
              , time_type
       from RISK_NHANNT.tv_fin_investordb_enr_ifrs_sas t
       where product_org = 'ALL'
       union all
       select items
              , ifrs_vas
              , company
              , product_org
              , monthid
              , CASE WHEN ITEMS IN ('TOTAL_NPL','TOTAL_WO/ANR') THEN amt_ ELSE AMT_/1000 END AS AMT_
              , time_type
       from RISK_NHANNT.tv_fin_investordb_enr_v_sas t
       where product_org = 'ALL'
       UNION ALL
       select case when ifrs_vas = 'IFRS' then 'ENR' || '_B' || bucket_
                   else 'ENR' || '_G' || bucket_ end as items
              , ifrs_vas
              , company
              , 'ALL' product_org
              , monthid
              , SUM(amt_)/1000 AMT_
              , time_type
       from RISK_NHANNT.TV_FIN_INVESTORDB_ENR_PROD_SAS
       group by case when ifrs_vas = 'IFRS' then 'ENR' || '_B' || bucket_
                   else 'ENR' || '_G' || bucket_ end
                , ifrs_vas
                , company
                , monthid
                , time_type;

## 3. Chart

       CREATE OR REPLACE VIEW V_FIN_PL_FC_EXT_MANAGEMENT_3_UNAUDIT AS
       with dtl as
       ( -- create table dtl as
       select ITEMS, (CAST (null AS VARCHAR2(320))) PRODUCT, MONTHID, VAL, TIME_TYPE
       from V_FIN_PL_FC_EXT_MANAGEMENT_3 -- FS actual
       where amt_type='MGMT_AMT_FINAL' and monthid >'201912'
       UNION ALL
       SELECT *
       FROM V_FIN_SALES_ENR_MOVEMENT -- Sales & ENR movement Actual
       WHERE ITEMS <> 'ENR'
       union all
       select ITEMS
              , CASE WHEN PRODUCT ='DCARD' THEN 'CDL'
                       WHEN PRODUCT = 'PL_CASH24' THEN 'PLNTB'
                         ELSE PRODUCT END AS PRODUCT
              , MONTHID
              , SUM(VALUE) VAL
              , TIME_TYPE
       from V_FIN_SALES_ENR_MOVEMENT t -- Sales & ENR movement Actual
       WHERE ITEMS='ENR'
       GROUP BY ITEMS
                , CASE WHEN PRODUCT ='DCARD' THEN 'CDL'
                         WHEN PRODUCT = 'PL_CASH24' THEN 'PLNTB'
                           ELSE PRODUCT END
                , MONTHID
                , TIME_TYPE
       )
       select k.items
              , k.product
              , k.monthid
              , k.val actual
              , k.time_type
              , t.value plan
              , k.val/t.value A_P
       from dtl k
       left join V_FIN_SALES_ENR_MOVEMENT_PLAN t on upper(t.items)=upper(k.items)
                                                 and (NVL(t.PRODUCT,0))=to_char(NVL(k.product,0))
                                                 and to_char(t.MONTHID)=to_char(k.monthid)
       WHERE K.TIME_TYPE <> '3.HALF_YEAR';

## 3. ENR GROUP

       create or replace view v_fin_enr_npl_by_product as
       with dtl as
       (
       select case when ifrs_vas = 'IFRS' then 'ENR' || '_B' || bucket_
                   else 'ENR' || '_G' || bucket_ end as items
              , ifrs_vas
              , company
              , case when product = 'TW' THEN '1.TW'
                       when product = 'CD' THEN '2.CD'
                         when product = 'PLNTB' THEN '3.PLNTB'
                           when product = 'PL_CASH24' THEN '4.PL_CASH24'
                             when product = 'TU' THEN '5.TU'
                               when product = 'XS' THEN '6.XS'
                                 when product = 'CRC' THEN '7.CRC'
                                   when product = 'DCARD' THEN '8.DCARD'
                                     when product = 'Syndicate_loan' THEN '9.SYNDICATE_LOAN'
                                       end as product_org
              , monthid
              , amt_/1000 AMT_
              , time_type
       from RISK_NHANNT.TV_FIN_INVESTORDB_ENR_PROD_SAS
       where product <> 'ALL'
       union all
       select items
              , ifrs_vas
              , company
              , product_org
              , monthid
              , CASE WHEN ITEMS IN ('NPL(%)') THEN amt_ ELSE AMT_/1000 END AS AMT_
              , time_type
       from RISK_NHANNT.tv_fin_investordb_enr_ifrs_sas t
       where items in ('ENR(0-6)','ENR(4-6)','NPL(%)')
       union all
       select items
              , ifrs_vas
              , company
              , product_org
              , monthid
              , CASE WHEN ITEMS IN ('NPL(%)') THEN amt_ ELSE AMT_/1000 END AS AMT_
              , time_type
       from RISK_NHANNT.tv_fin_investordb_enr_v_sas t
       where items in ('ENR(1-5)','ENR(3-5)','NPL(%)')
       )
       select t.product_org
              , k.items
              , ifrs_vas
              , company
              , monthid
              , amt_
              , time_type
       from impt_list_of_product t
       left join dtl k on t.product_org=k.product_org;
 
