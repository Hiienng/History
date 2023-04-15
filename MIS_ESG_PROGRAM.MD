# REFRESHABLE REPORT FOR ESG PROGRAME

       create or replace view v_esg_data as
       with General_customer as
       (select RPT_YEAR
              , 'All status' PRIORITY
              , case when substr(rpt_month,5,2) LIKE '03' then 'Q1'
                   when  substr(rpt_month,5,2) like '06' then 'Q2'
                     when  substr(rpt_month,5,2) like '09' then 'Q3'
                       when  substr(rpt_month,5,2) like '12' then 'Q4'
                      else null end as RPT_Quarter
              , SUM(No_of_cust) Value_
              , case when (SUB_PRIORITY_VALUE like 'FEMALE' OR SUB_PRIORITY_VALUE like 'UNKNOWN') THEN 'FEMALE'
              ELSE 'MALE' END AS SUB_PRIORITY_VALUE
       from MIS.V_CEOO_PERIODIC_CUSBASE_QL t
       where upper(sub_priority) like '%GENDER%'
       and SUB_PRIORITY_VALUE not like 'MALE'
       and substr(rpt_month,5,2) in ('03','06','09','12')
       GROUP BY  RPT_YEAR
                 , case when substr(rpt_month,5,2) LIKE '03' then 'Q1'
                   when  substr(rpt_month,5,2) like '06' then 'Q2'
                     when  substr(rpt_month,5,2) like '09' then 'Q3'
                       when  substr(rpt_month,5,2) like '12' then 'Q4'
                      else null end
                , case when (SUB_PRIORITY_VALUE like 'FEMALE' OR SUB_PRIORITY_VALUE like 'UNKNOWN') THEN 'FEMALE'
              ELSE 'MALE' END
       union all
       select RPT_YEAR
              , 'All status' PRIORITY
              , case when substr(rpt_month,5,2) LIKE '03' then 'Q1'
                   when  substr(rpt_month,5,2) like '06' then 'Q2'
                     when  substr(rpt_month,5,2) like '09' then 'Q3'
                       when  substr(rpt_month,5,2) like '12' then 'Q4'
                      else null end as RPT_Quarter
              , SUM(No_of_cust) Value_
              , 'ALL GEN' SUB_PRIORITY_VALUE
       from MIS.V_CEOO_PERIODIC_CUSBASE_QL t
       where upper(sub_priority) like '%GENDER%' and substr(rpt_month,5,2) in ('03','06','09','12')
       GROUP BY  RPT_YEAR
                 , case when substr(rpt_month,5,2) LIKE '03' then 'Q1'
                   when  substr(rpt_month,5,2) like '06' then 'Q2'
                     when  substr(rpt_month,5,2) like '09' then 'Q3'
                       when  substr(rpt_month,5,2) like '12' then 'Q4'
                      else null end
       order by rpt_year, RPT_Quarter
       )
       ,ACTIVE_customer as
       ( select RPT_YEAR
              , PRIORITY
              , case when substr(rpt_month,5,2) LIKE '03' then 'Q1'
                   when  substr(rpt_month,5,2) like '06' then 'Q2'
                     when  substr(rpt_month,5,2) like '09' then 'Q3'
                       when  substr(rpt_month,5,2) like '12' then 'Q4'
                      else null end as RPT_Quarter
              , SUM(No_of_cust) Value_
              , case when (SUB_PRIORITY_VALUE like 'FEMALE' OR SUB_PRIORITY_VALUE like 'UNKNOWN') THEN 'FEMALE'
              ELSE 'MALE' END AS SUB_PRIORITY_VALUE
       from MIS.V_CEOO_PERIODIC_CUSBASE_QL t
       where upper(sub_priority) like '%GENDER%'
       and SUB_PRIORITY_VALUE not like 'MALE'
       AND PRIORITY LIKE '1.Active customers'
       and substr(rpt_month,5,2) in ('03','06','09','12')
       GROUP BY  RPT_YEAR
                 , PRIORITY
                 , case when substr(rpt_month,5,2) LIKE '03' then 'Q1'
                   when  substr(rpt_month,5,2) like '06' then 'Q2'
                     when  substr(rpt_month,5,2) like '09' then 'Q3'
                       when  substr(rpt_month,5,2) like '12' then 'Q4'
                      else null end
                , case when (SUB_PRIORITY_VALUE like 'FEMALE' OR SUB_PRIORITY_VALUE like 'UNKNOWN') THEN 'FEMALE'
              ELSE 'MALE' END
       union all
       select RPT_YEAR
              , PRIORITY
              , case when substr(rpt_month,5,2) LIKE '03' then 'Q1'
                   when  substr(rpt_month,5,2) like '06' then 'Q2'
                     when  substr(rpt_month,5,2) like '09' then 'Q3'
                       when  substr(rpt_month,5,2) like '12' then 'Q4'
                      else null end as RPT_Quarter
              , SUM(No_of_cust) Value_
              , 'ALL GEN' SUB_PRIORITY_VALUE
       from MIS.V_CEOO_PERIODIC_CUSBASE_QL t
       where upper(sub_priority) like '%GENDER%'
       AND PRIORITY LIKE '1.Active customers'
       and substr(rpt_month,5,2) in ('03','06','09','12')
       GROUP BY  RPT_YEAR
                 , PRIORITY
                 , case when substr(rpt_month,5,2) LIKE '03' then 'Q1'
                   when  substr(rpt_month,5,2) like '06' then 'Q2'
                     when  substr(rpt_month,5,2) like '09' then 'Q3'
                       when  substr(rpt_month,5,2) like '12' then 'Q4'
                      else null end
       )
       ,no_loans as
       (select to_number(substr(monthid,1,4)) RPT_YEAR -- chuy?n d?i d? li?u thành number
              , 'Total sales' PRIORITY
              , case when substr(monthid,5,2) in ('01','02','03') then 'Q1'
                when substr(monthid,5,2) in ('04','05','06') then 'Q2'
                  when substr(monthid,5,2) in ('07','08','09') then 'Q3'
                    else 'Q4' end as RPT_Quarter
              , SUM(amt_)/1000 Value_
              , 'Billion' SUB_PRIORITY_VALUE
       from  MIS.TV_FIN_INVESTORDB_SALEDISB_SAS
       where upper(items) like '%TOTAL%' and time_type like 'MONTHLY'
       group by substr(monthid,1,4)
                ,  case when substr(monthid,5,2) in ('01','02','03') then 'Q1'
                when substr(monthid,5,2) in ('04','05','06') then 'Q2'
                  when substr(monthid,5,2) in ('07','08','09') then 'Q3'
                    else 'Q4' end
       )
       , ROBO_ as
       (select to_number(substr(RPT_MONTH,1,4)) RPT_YEAR
               , 'Number underserved cust' PRIORITY
               , case when substr(RPT_MONTH,6,2) in ('01','02','03') then 'Q1'
                when substr(RPT_MONTH,6,2) in ('04','05','06') then 'Q2'
                  when substr(RPT_MONTH,6,2) in ('07','08','09') then 'Q3'
                    else 'Q4' end AS RPT_Quarter
               , cus_reject_1M Value_
               , NULL SUB_PRIORITY_VALUE
       from V_CEOO_DB_CUS_REJECT_QL_V2
       )
       Select "RPT_YEAR","PRIORITY","RPT_QUARTER","VALUE_","SUB_PRIORITY_VALUE" from General_customer
       union all
       select "RPT_YEAR","PRIORITY","RPT_QUARTER","VALUE_","SUB_PRIORITY_VALUE" from ACTIVE_customer
       union all
       select "RPT_YEAR","PRIORITY","RPT_QUARTER","VALUE_","SUB_PRIORITY_VALUE" from ROBO_
       union all
       select "RPT_YEAR","PRIORITY","RPT_QUARTER","VALUE_","SUB_PRIORITY_VALUE" from no_loans
       ;
