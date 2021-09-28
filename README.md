# First-real-one-Dual-close-report
Count of accounts by  request_id that have dual commodities (Gas&amp;Electric)


with Gas as (
  select
    executed_date ,
    request_id ,
    account_number ,
    commodity,
    r.customer_name,
    r.status_id  as bgs
  from commercial_base cb1
  left join eligo_pricing_requests r
         on  r.id = cb1.request_id
  where cb1.commodity = 'G'
            and cb1.sales_channel_name = 'Inside Sales'
            and cb1.executed_date >= current_date - interval '365 days'
),
Electric as (
      select
        executed_date ,
        request_id ,
        account_number ,
        commodity
      from commercial_base cb1
    where cb1.commodity = 'E'
            and cb1.sales_channel_name = 'Inside Sales'
            and cb1.executed_date >= current_date - interval '365 days'
  ),
  dual as (select
             distinct
             Gas.request_id                        a,
             date_trunc('week', gas.executed_date) b,
             gas.commodity                         c
              --electric.commodity                   j

           from gas
                full outer join electric
               on Gas.request_id = Electric.request_id
                  and Gas.executed_date = Electric.executed_date
             left join eligo_pricing_statuses s on s.id = bgs
             --where gas.request_id = null
    group by 1,2,3
  ),

  together as (
      select
        dual.*,
        cb1.account_number                     acc_num,
        cb1.request_id                        d,
        date_trunc('week', cb1.executed_date) e,
        min(cb1.commodity)                    f


      from dual
        left outer join commercial_base cb1 on cb1.request_id = dual.a
                                         and cb1.sales_channel_name = 'Inside Sales'
                                         and cb1.executed_date >= current_date - interval '365 days'
      group by 1, 2, 3, 4, 5, 6
  )
--select*from together
--order by together.d;
  select together.acc_num,together.d, coalesce(b,e),
  count(case when f = 'E' and c = 'G'  then 1 else null end )over (partition by d) as "Dual",
  count(case when f = 'E' and f != 'G'  then 1 else null end )over (partition by d) as "E",

  count(case when f = 'G' and c = 'G'  then 1 else null end )over (partition by d) as "G"
from together
where e is not null
group by coalesce(b,e), together.f, together.a, together.d, together.acc_num, together.d, together.c
order by coalesce(b,e);


--select together.acc_num,together.d, coalesce(b,e),
  --count(case when f = 'E' and a is not null then 1 else null end )over (partition by d) as "Dual",
  --count(case when f = 'E' and a is null then 1 else null end )over (partition by d) as "E",
  --count(case when f = 'G' and a is not null then 1 else null end )over (partition by d) as "G"
--from together
--where e is not null
--group by coalesce(b,e), together.f, together.a, together.d, together.acc_num, together.d
--order by coalesce(b,e);
--The main difference here between this query and the 'First try on Dual' query is there is a partition clause on the count agg functions to find how many accounts belong to a singular request_id
-- The main issue here is that I can not seem to fill the "E" column, seemingly because column a from Dual does not include null-request_id's. No matter the type of join A seemingly only has gas request_id's. I need to find a way to have the Dual request_id include those that do not match.
