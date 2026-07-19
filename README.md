# sql-code

## examples of sql code
### 1
    select EventID,
           EventName,
           case
               when EventLabel in
                    (
                     'prostornye_kvartiry_ot_75_m',
                     'prostornye_kvartiry_ot_75_m_'
                        ) then 'prostornye_kvartiry_ot_75_m'
               else EventLabel end  event_label,
           EventAction,
           case
               when EventID in ('58', '59', '60', '1177', '96', '1259') then 'any'
               else
                   EventContent end event_content,
           case
               when EventID in ('58', '59', '60', '1177', '96', '1259') then 'any'
               else
                   EventContext end event_context,
           count(1)                 cnt,
           count(distinct VisitID)  visit_cnt
    from Level_Site.YM_Events
    where "Date" >= '2025-12-30'
      and screenName = '/premium/'
      and EventID in ('1249', '1248')
    group by EventID, EventName, event_label, EventAction, event_content, event_context
    order by visit_cnt desc;
### 2

    select
        --EventLabel,
           count(1)                                                              cnt,
           count(distinct VisitID)                                               visit_cnt
    from Level_Site.YM_Events
    where "Date" >= '2025-12-30'
      and screenName = '/premium/'
      and EventLabel in ('smotret_kvartiry', 'smotret_vse_kvartiry')
    --group by EventLabel
    order by visit_cnt desc;
    
    with visits_for_calls as (select ClientID,
                                     VisitID::Nullable(String) visit_id,
                                     min("DateTime")           start_dt,
                                     max("DateTime")           end_dt
                              from Level_Site.YM_Events
                              where "Date" >= '2025-10-21'
                              group by ClientID, VisitID),
         calls as (select start_time,
                          ym_client_id,
                          case when attributes like '%through%' then 'spk' else 'not_spk' end is_spk
                   from Level_Site.Comagic_Calls
                   where communication_page_url like 'https://level.ru%'
                     and ym_client_id is not null
                     and start_time >= '2025-10-21'),
         calls_visits_raw as (select c.start_time,
                                     c.ym_client_id,
                                     c.is_spk,
                                     v.visit_id,
                                     v.start_dt,
                                     v.end_dt,
                                     row_number() over (partition by c.start_time, c.ym_client_id order by v.start_dt) rn
                              from calls c
                                       left join visits_for_calls v on c.ym_client_id::String = v.ClientID::String and
                                                                       c.start_time::date = v.start_dt::date
                              order by c.start_time desc),
         calls_visits as (select visit_id,
                                 count(1)                                   call_cnt,
                                 count(case when is_spk = 'spk' then 1 end) spk_call_cnt
                          from calls_visits_raw
                          where rn = 1
                            and visit_id is not null
                          group by visit_id),
         visits_raw as (select "Date"::Nullable(Date)                                         d,
                               VisitID,
                               max(nullIf(toString(LastTrafficSource), ''))::Nullable(String) last_traffic_source,
                               max(nullIf(toString(LastUTMSource), ''))::Nullable(String)     last_utm_source,
                               max(nullIf(toString(LastUTMCampaign), ''))::Nullable(String)   last_campaign,
                               count(case when EventName = 'page_view' then 1 end)            page_views,
                               count(case
                                         when EventName = 'page_view' and
                                              (screenName like '%/flat/%' or screenName like '%apartment%')
                                             then 1 end)                                      lot_page_views,
                               count(case when EventID in ('1167', '17') then 1 end)          appointment_click_cnt,
                               count(case
                                         when EventAction = 'button_click' and EventLabel = 'zakazat_zvonok'
                                             then 1 end)                                      call_order_click_cnt,
                               max("DateTime") - min("DateTime")                              duration
                        from Level_Site.YM_Events
                        where "Date" >= '2025-10-21'
                        group by d, VisitID),
         ym_visits as (select d,
                              v.VisitID,
                              v.last_traffic_source,
                              v.last_utm_source,
                              v.last_campaign,
                              v.page_views,
                              v.lot_page_views,
                              v.duration,
                              v.appointment_click_cnt,
                              v.call_order_click_cnt,
                              cv.call_cnt,
                              cv.spk_call_cnt
                       from visits_raw v
                                left join calls_visits cv on v.VisitID::String = cv.visit_id::String),
         vis as (select VisitID,
                        screenName,
                        "DateTime"                                                   dt,
                        "Date"                                                       d,
                        row_number() over (partition by VisitID order by "DateTime") rn
                 from Level_Site.YM_Events
                 where "Date" >= '2025-10-21'),
         visits as (select case
                               when
                                   screenName = '/premium/' and d between '2025-10-21' and '2025-12-29'
                                   then 'premium-collection'
                               when screenName = '/premium/' and d between '2025-12-30' and '2026-04-28'
                                   then 'premium-landing'
                               else 'other' end as visit_type,
                           *
                    from vis
                    where rn = 1),
         final_visits as (select y.d,
                                 y.VisitID,
                                 y.page_views,
                                 y.lot_page_views,
                                 y.duration,
                                 y.call_cnt,
                                 y.spk_call_cnt,
                                 y.appointment_click_cnt,
                                 y.call_order_click_cnt,
                                 v.visit_type
                          from ym_visits y
                                   join visits v on y.VisitID = v.VisitID)
    select visit_type,
           count(distinct VisitID)    visit_cnt,
           sum(page_views)            sum_page_views,
           sum(lot_page_views)        sum_lot_page_views,
           sum(duration)              sum_duration,
           sum(call_cnt)              sum_call_cnt,
           sum(spk_call_cnt)          sum_spk_call_cnt,
           sum(appointment_click_cnt) sum_appointment_click_cnt,
           sum(call_order_click_cnt)  sum_call_order_click_cnt
    from final_visits
    group by visit_type
    order by visit_cnt desc
    limit 1000;


### 3

