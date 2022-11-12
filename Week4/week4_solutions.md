**Schema (PostgreSQL v13)**

---
<p align=center><b>A. Customer Nodes Exploration</b>

---

**Query #1** How many unique nodes are there on the Data Bank system?

    SELECT COUNT(DISTINCT(node_id)) as unique_nodes
    FROM data_bank.customer_nodes;

| unique_nodes |
| ------------ |
| 5            |

---
**Query #2** What is the number of nodes per region?

    SELECT region_id, COUNT(DISTINCT(node_id)) node_count
    FROM data_bank.customer_nodes
    GROUP BY region_id;

| region_id | node_count |
| --------- | ---------- |
| 1         | 5          |
| 2         | 5          |
| 3         | 5          |
| 4         | 5          |
| 5         | 5          |

---
**Query #3** How many customers are allocated to each region?

    SELECT region_id, COUNT(DISTINCT(customer_id)) as customer_count
    FROM data_bank.customer_nodes
    GROUP BY region_id;

| region_id | customer_count |
| --------- | -------------- |
| 1         | 110            |
| 2         | 105            |
| 3         | 102            |
| 4         | 95             |
| 5         | 88             |

---
**Query #4** How many days on average are customers reallocated to a different node?

    SELECT AVG(end_date - start_date) as average_days FROM data_bank.customer_nodes;

| average_days        |
| ------------------- |
| 416373.411714285714 |

---
**Query #5** What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

    SELECT region_id, PERCENTILE_CONT(0.5) WITHIN GROUP(ORDER BY end_date - start_date) as median_percentile_50, PERCENTILE_CONT(0.8) WITHIN GROUP(ORDER BY end_date - start_date) as percentile_80, PERCENTILE_CONT(0.95) WITHIN GROUP(ORDER BY end_date - start_date) as percentile_95 FROM data_bank.customer_nodes
    GROUP BY region_id;

| region_id | median_percentile_50 | percentile_80 | percentile_95 |
| --------- | -------------------- | ------------- | ------------- |
| 1         | 17                   | 28            | 2914533.55    |
| 2         | 18                   | 27            | 2914534.3     |
| 3         | 17.5                 | 27            | 2914535.35    |
| 4         | 17                   | 27            | 2914538       |
| 5         | 18                   | 28            | 2914527       |

---
<p align=center><b>B. Customer Transactions</b>

---
**Query #1** What is the unique count and total amount for each transaction type?

    SELECT txn_type, COUNT(txn_amount) as unique_count, SUM(txn_amount) as total_amount 
    FROM data_bank.customer_transactions
    GROUP BY txn_type;

| txn_type   | unique_count | total_amount |
| ---------- | ------------ | ------------ |
| purchase   | 1617         | 806537       |
| deposit    | 2671         | 1359168      |
| withdrawal | 1580         | 793003       |

---
**Query #2** What is the average total historical deposit counts and amounts for all customers?

    SELECT COUNT(*) as deposit_counts, AVG(txn_amount) as average_amounts
    FROM data_bank.customer_transactions
    WHERE txn_type = 'deposit';

| deposit_counts | average_amounts      |
| -------------- | -------------------- |
| 2671           | 508.8611007113440659 |

---
**Query #3** For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

    WITH txn_cte AS
    (SELECT EXTRACT(MONTH FROM txn_date) as month_num, TO_CHAR(txn_date, 'Month') as month_name, customer_id,
     SUM(CASE
         	WHEN txn_type = 'deposit' THEN 1
         	ELSE 0
        END) as deposit_count,
     SUM(CASE
         	WHEN txn_type = 'purchase' THEN 1
         	ELSE 0
        END) as purchase_count,
     SUM(CASE
         	WHEN txn_type = 'withdrawal' THEN 1
         	ELSE 0
        END) as withdrawal_count
    FROM data_bank.customer_transactions
    GROUP BY month_num, month_name, customer_id
    ORDER BY month_num, customer_id)
    
    SELECT month_name, COUNT(*) 
    FROM txn_cte
    WHERE deposit_count > 1 AND
    (purchase_count = 1 OR withdrawal_count = 1)
    GROUP BY month_num, month_name
    ORDER BY month_num;

| month_name | count |
| ---------- | ----- |
| January    | 115   |
| February   | 108   |
| March      | 113   |
| April      | 50    |

---
**Query #4** What is the closing balance for each customer at the end of the month?

    WITH txn_cte AS
    (SELECT customer_id, EXTRACT(MONTH FROM txn_date) as month_num, TO_CHAR(txn_date, 'Month') as month_name,
     SUM(CASE
         	WHEN txn_type = 'deposit' THEN txn_amount
         	ELSE 0
        END) as deposit_amt,
     SUM(CASE
         	WHEN txn_type = 'purchase' THEN txn_amount
         	ELSE 0
        END) as purchase_amt,
     SUM(CASE
         	WHEN txn_type = 'withdrawal' THEN txn_amount
         	ELSE 0
        END) as withdrawal_amt
    FROM data_bank.customer_transactions
    GROUP BY customer_id, month_num, month_name
    ORDER BY customer_id, month_num)
    
    SELECT customer_id, month_name, SUM(deposit_amt - purchase_amt - withdrawal_amt) OVER(PARTITION BY customer_id ORDER BY month_num) AS closing_balance
    FROM txn_cte
    ORDER BY customer_id, month_num;

| customer_id | month_name | closing_balance |
| ----------- | ---------- | --------------- |
| 1           | January    | 312             |
| 1           | March      | -640            |
| 2           | January    | 549             |
| 2           | March      | 610             |
| 3           | January    | 144             |
| 3           | February   | -821            |
| 3           | March      | -1222           |
| 3           | April      | -729            |
| 4           | January    | 848             |
| 4           | March      | 655             |
| 5           | January    | 954             |
| 5           | March      | -1923           |
| 5           | April      | -2413           |
| 6           | January    | 733             |
| 6           | February   | -52             |
| 6           | March      | 340             |
| 7           | January    | 964             |
| 7           | February   | 3173            |
| 7           | March      | 2533            |
| 7           | April      | 2623            |
| 8           | January    | 587             |
| 8           | February   | 407             |
| 8           | March      | -57             |
| 8           | April      | -1029           |
| 9           | January    | 849             |
| 9           | February   | 654             |
| 9           | March      | 1584            |
| 9           | April      | 862             |
| 10          | January    | -1622           |
| 10          | February   | -1342           |
| 10          | March      | -2753           |
| 10          | April      | -5090           |
| 11          | January    | -1744           |
| 11          | February   | -2469           |
| 11          | March      | -2088           |
| 11          | April      | -2416           |
| 12          | January    | 92              |
| 12          | March      | 295             |
| 13          | January    | 780             |
| 13          | February   | 1279            |
| 13          | March      | 1405            |
| 14          | January    | 205             |
| 14          | February   | 821             |
| 14          | April      | 989             |
| 15          | January    | 379             |
| 15          | April      | 1102            |
| 16          | January    | -1341           |
| 16          | February   | -2893           |
| 16          | March      | -4284           |
| 16          | April      | -3422           |
| 17          | January    | 465             |
| 17          | February   | -892            |
| 18          | January    | 757             |
| 18          | February   | -424            |
| 18          | March      | -842            |
| 18          | April      | -815            |
| 19          | January    | -12             |
| 19          | February   | -251            |
| 19          | March      | -301            |
| 19          | April      | 42              |
| 20          | January    | 465             |
| 20          | February   | 519             |
| 20          | March      | 776             |
| 21          | January    | -204            |
| 21          | February   | -764            |
| 21          | March      | -1874           |
| 21          | April      | -3253           |
| 22          | January    | 235             |
| 22          | February   | -1039           |
| 22          | March      | -149            |
| 22          | April      | -1358           |
| 23          | January    | 94              |
| 23          | February   | -314            |
| 23          | March      | -156            |
| 23          | April      | -678            |
| 24          | January    | 615             |
| 24          | February   | 813             |
| 24          | March      | 254             |
| 25          | January    | 174             |
| 25          | February   | -400            |
| 25          | March      | -1220           |
| 25          | April      | -304            |
| 26          | January    | 638             |
| 26          | February   | -31             |
| 26          | March      | -622            |
| 26          | April      | -1870           |
| 27          | January    | -1189           |
| 27          | February   | -713            |
| 27          | March      | -3116           |
| 28          | January    | 451             |
| 28          | February   | -818            |
| 28          | March      | -1228           |
| 28          | April      | 272             |
| 29          | January    | -138            |
| 29          | February   | -76             |
| 29          | March      | 831             |
| 29          | April      | -548            |
| 30          | January    | 33              |
| 30          | February   | -431            |
| 30          | April      | 508             |
| 31          | January    | 83              |
| 31          | March      | -141            |
| 32          | January    | -89             |
| 32          | February   | 376             |
| 32          | March      | -843            |
| 32          | April      | -1001           |
| 33          | January    | 473             |
| 33          | February   | -116            |
| 33          | March      | 1225            |
| 33          | April      | 989             |
| 34          | January    | 976             |
| 34          | February   | -347            |
| 34          | March      | -185            |
| 35          | January    | 507             |
| 35          | February   | -821            |
| 35          | March      | -1163           |
| 36          | January    | 149             |
| 36          | February   | 290             |
| 36          | March      | 1041            |
| 36          | April      | 427             |
| 37          | January    | 85              |
| 37          | February   | 902             |
| 37          | March      | -1069           |
| 37          | April      | -959            |
| 38          | January    | 367             |
| 38          | February   | -465            |
| 38          | March      | -798            |
| 38          | April      | -1246           |
| 39          | January    | 1429            |
| 39          | February   | 2388            |
| 39          | March      | 2460            |
| 39          | April      | 2516            |
| 40          | January    | 347             |
| 40          | February   | 295             |
| 40          | March      | 659             |
| 40          | April      | -208            |
| 41          | January    | -46             |
| 41          | February   | 1379            |
| 41          | March      | 3441            |
| 41          | April      | 2525            |
| 42          | January    | 447             |
| 42          | February   | 1067            |
| 42          | March      | -887            |
| 42          | April      | -1886           |
| 43          | January    | -201            |
| 43          | February   | -406            |
| 43          | March      | 869             |
| 43          | April      | 545             |
| 44          | January    | -690            |
| 44          | February   | -19             |
| 44          | April      | -339            |
| 45          | January    | 940             |
| 45          | February   | -1152           |
| 45          | March      | 584             |
| 46          | January    | 522             |
| 46          | February   | 1388            |
| 46          | March      | 80              |
| 46          | April      | 104             |
| 47          | January    | -1153           |
| 47          | February   | -1283           |
| 47          | March      | -2862           |
| 47          | April      | -3169           |
| 48          | January    | -2368           |
| 48          | February   | -2972           |
| 48          | March      | -3745           |
| 49          | January    | -397            |
| 49          | February   | -594            |
| 49          | March      | -2556           |
| 50          | January    | 931             |
| 50          | February   | -674            |
| 50          | March      | 275             |
| 50          | April      | 450             |
| 51          | January    | 301             |
| 51          | February   | -97             |
| 51          | March      | 779             |
| 51          | April      | 1364            |
| 52          | January    | 1140            |
| 52          | February   | 2612            |
| 53          | January    | 22              |
| 53          | February   | 210             |
| 53          | March      | -728            |
| 53          | April      | 227             |
| 54          | January    | 1658            |
| 54          | February   | 1629            |
| 54          | March      | 533             |
| 54          | April      | 968             |
| 55          | January    | 380             |
| 55          | February   | -410            |
| 55          | March      | 349             |
| 55          | April      | -513            |
| 56          | January    | -67             |
| 56          | February   | -1646           |
| 56          | March      | -2075           |
| 56          | April      | -3866           |
| 57          | January    | 414             |
| 57          | February   | -101            |
| 57          | March      | -866            |
| 58          | January    | 383             |
| 58          | February   | 1697            |
| 58          | March      | -1196           |
| 58          | April      | -635            |
| 59          | January    | 924             |
| 59          | February   | 2190            |
| 59          | March      | 1652            |
| 59          | April      | 798             |
| 60          | January    | -189            |
| 60          | February   | 668             |
| 60          | March      | -745            |
| 60          | April      | -1169           |
| 61          | January    | 222             |
| 61          | February   | 323             |
| 61          | March      | -1710           |
| 61          | April      | -2237           |
| 62          | January    | -212            |
| 62          | March      | -763            |
| 63          | January    | -332            |
| 63          | February   | -953            |
| 63          | March      | -3946           |
| 64          | January    | 2332            |
| 64          | February   | 1554            |
| 64          | March      | 245             |
| 65          | January    | 25              |
| 65          | March      | -450            |
| 65          | April      | -1381           |
| 66          | January    | 1971            |
| 66          | February   | 1492            |
| 66          | March      | -517            |
| 67          | January    | 1593            |
| 67          | February   | 2565            |
| 67          | March      | 2050            |
| 67          | April      | 1222            |
| 68          | January    | 574             |
| 68          | February   | 278             |
| 68          | March      | -456            |
| 69          | January    | 23              |
| 69          | February   | -1944           |
| 69          | March      | -2338           |
| 69          | April      | -3085           |
| 70          | January    | -584            |
| 70          | February   | -63             |
| 70          | March      | -1814           |
| 71          | January    | 128             |
| 71          | February   | -673            |
| 71          | March      | -1265           |
| 72          | January    | 796             |
| 72          | February   | -803            |
| 72          | March      | -1680           |
| 72          | April      | -2327           |
| 73          | January    | 513             |
| 74          | January    | 229             |
| 74          | March      | 318             |
| 75          | January    | 234             |
| 75          | February   | 294             |
| 76          | January    | 925             |
| 76          | February   | 2081            |
| 76          | March      | 435             |
| 77          | January    | 120             |
| 77          | February   | 501             |
| 77          | March      | 797             |
| 78          | January    | 694             |
| 78          | February   | -762            |
| 78          | March      | -717            |
| 78          | April      | -976            |
| 79          | January    | 521             |
| 79          | February   | 1380            |
| 80          | January    | 795             |
| 80          | February   | 1190            |
| 80          | March      | 622             |
| 80          | April      | 199             |
| 81          | January    | 403             |
| 81          | February   | -957            |
| 81          | March      | -1106           |
| 81          | April      | -1984           |
| 82          | January    | -3912           |
| 82          | February   | -3986           |
| 82          | March      | -3249           |
| 82          | April      | -4614           |
| 83          | January    | 1099            |
| 83          | February   | -692            |
| 83          | March      | -742            |
| 83          | April      | -377            |
| 84          | January    | 968             |
| 84          | March      | 609             |
| 85          | January    | 467             |
| 85          | March      | 1076            |
| 85          | April      | 646             |
| 86          | January    | 872             |
| 86          | February   | -504            |
| 86          | March      | 93              |
| 87          | January    | -365            |
| 87          | February   | -1366           |
| 87          | March      | -1563           |
| 87          | April      | -1195           |
| 88          | January    | -35             |
| 88          | February   | 752             |
| 88          | March      | -736            |
| 88          | April      | -820            |
| 89          | January    | 210             |
| 89          | February   | -1679           |
| 89          | March      | -2653           |
| 89          | April      | -3147           |
| 90          | January    | 1772            |
| 90          | February   | -1235           |
| 90          | March      | -1624           |
| 90          | April      | -1846           |
| 91          | January    | -47             |
| 91          | February   | -959            |
| 91          | March      | -2660           |
| 91          | April      | -2495           |
| 92          | January    | 985             |
| 92          | March      | 142             |
| 93          | January    | 399             |
| 93          | February   | 1103            |
| 93          | March      | 1186            |
| 93          | April      | 968             |
| 94          | January    | -766            |
| 94          | February   | -1496           |
| 94          | March      | -1542           |
| 95          | January    | 217             |
| 95          | February   | 960             |
| 95          | March      | 1446            |
| 96          | January    | 1048            |
| 96          | February   | 1537            |
| 96          | March      | 942             |
| 97          | January    | 623             |
| 97          | February   | -240            |
| 97          | March      | -2483           |
| 98          | January    | 622             |
| 98          | February   | 287             |
| 98          | March      | -95             |
| 98          | April      | 750             |
| 99          | January    | 949             |
| 99          | February   | 760             |
| 99          | March      | 737             |
| 100         | January    | 1081            |
| 100         | February   | -497            |
| 100         | March      | -1451           |
| 101         | January    | -484            |
| 101         | February   | -1324           |
| 101         | March      | -2673           |
| 102         | January    | 917             |
| 102         | February   | 1428            |
| 102         | March      | 1865            |
| 102         | April      | 646             |
| 103         | January    | 240             |
| 103         | February   | -850            |
| 103         | March      | -2257           |
| 104         | January    | 615             |
| 104         | February   | 1087            |
| 104         | March      | 1190            |
| 105         | January    | 1014            |
| 105         | February   | 166             |
| 105         | March      | 27              |
| 105         | April      | -186            |
| 106         | January    | -109            |
| 106         | February   | 846             |
| 106         | March      | -111            |
| 106         | April      | -1462           |
| 107         | January    | -144            |
| 107         | February   | -690            |
| 108         | January    | 530             |
| 108         | February   | 738             |
| 108         | March      | 1546            |
| 108         | April      | 2680            |
| 109         | January    | 429             |
| 109         | February   | 2491            |
| 110         | January    | 1258            |
| 110         | February   | 1198            |
| 110         | March      | 2233            |
| 111         | January    | 101             |
| 111         | February   | 463             |
| 111         | March      | 99              |
| 112         | January    | 945             |
| 112         | February   | 893             |
| 112         | March      | -116            |
| 113         | January    | -511            |
| 113         | February   | 62              |
| 113         | March      | 12              |
| 113         | April      | -1140           |
| 114         | January    | 743             |
| 114         | March      | 169             |
| 114         | April      | 1143            |
| 115         | January    | 144             |
| 115         | February   | -845            |
| 115         | March      | 884             |
| 115         | April      | -41             |
| 116         | January    | 167             |
| 116         | February   | 53              |
| 116         | March      | 543             |
| 116         | April      | 330             |
| 117         | January    | -25             |
| 117         | February   | -216            |
| 117         | March      | -706            |
| 118         | January    | -683            |
| 118         | February   | -513            |
| 118         | March      | -1427           |
| 119         | January    | 62              |
| 119         | March      | -907            |
| 119         | April      | -490            |
| 120         | January    | 824             |
| 120         | February   | 1913            |
| 120         | March      | -900            |
| 120         | April      | -1465           |
| 121         | January    | 1992            |
| 121         | February   | 1296            |
| 121         | March      | -425            |
| 122         | January    | 314             |
| 122         | February   | 252             |
| 122         | March      | 1347            |
| 122         | April      | 1066            |
| 123         | January    | -717            |
| 123         | February   | -2277           |
| 123         | March      | -1584           |
| 123         | April      | -2128           |
| 124         | January    | 731             |
| 124         | February   | 1878            |
| 124         | March      | 1301            |
| 125         | January    | -791            |
| 125         | February   | -2479           |
| 125         | March      | -2436           |
| 126         | January    | -786            |
| 126         | February   | -716            |
| 126         | March      | -2822           |
| 127         | January    | 217             |
| 127         | February   | 703             |
| 127         | April      | 1672            |
| 128         | January    | 410             |
| 128         | February   | 144             |
| 128         | March      | -776            |
| 128         | April      | -202            |
| 129         | January    | 466             |
| 129         | February   | -796            |
| 129         | March      | 68              |
| 129         | April      | -2007           |
| 130         | January    | -248            |
| 130         | February   | -1160           |
| 130         | March      | 132             |
| 131         | January    | 480             |
| 131         | February   | -983            |
| 131         | March      | -152            |
| 132         | January    | -1254           |
| 132         | February   | -2844           |
| 132         | March      | -5256           |
| 132         | April      | -5585           |
| 133         | January    | -356            |
| 133         | February   | -368            |
| 134         | January    | 3194            |
| 134         | February   | 2748            |
| 134         | March      | 1768            |
| 135         | January    | 104             |
| 135         | February   | 977             |
| 135         | March      | 1001            |
| 136         | January    | 479             |
| 136         | February   | 966             |
| 136         | March      | 383             |
| 136         | April      | -133            |
| 137         | January    | 396             |
| 137         | February   | -356            |
| 138         | January    | 1316            |
| 138         | February   | 320             |
| 138         | March      | 75              |
| 138         | April      | -775            |
| 139         | January    | 44              |
| 139         | February   | 504             |
| 139         | March      | 537             |
| 140         | January    | 803             |
| 140         | February   | 1526            |
| 140         | March      | 2345            |
| 140         | April      | 1495            |
| 141         | January    | -369            |
| 141         | February   | 1483            |
| 141         | March      | 2113            |
| 141         | April      | 2538            |
| 142         | January    | 1378            |
| 142         | February   | 440             |
| 142         | March      | 217             |
| 142         | April      | 863             |
| 143         | January    | 807             |
| 143         | February   | 1625            |
| 143         | March      | 26              |
| 143         | April      | -2457           |
| 144         | January    | -735            |
| 144         | February   | -3280           |
| 144         | March      | -3046           |
| 144         | April      | -4395           |
| 145         | January    | -3051           |
| 145         | February   | -1970           |
| 145         | March      | -4119           |
| 146         | January    | -807            |
| 146         | February   | -3267           |
| 146         | March      | -3781           |
| 146         | April      | -3717           |
| 147         | January    | 600             |
| 147         | February   | 1698            |
| 148         | January    | 88              |
| 148         | February   | -2467           |
| 148         | March      | -2076           |
| 148         | April      | -2730           |
| 149         | January    | 344             |
| 149         | February   | 321             |
| 149         | March      | -202            |
| 150         | January    | -600            |
| 150         | February   | -1512           |
| 150         | March      | -1604           |
| 150         | April      | -2429           |
| 151         | January    | 1367            |
| 151         | February   | 5               |
| 151         | March      | -887            |
| 152         | January    | 1831            |
| 152         | February   | 1902            |
| 152         | March      | 1984            |
| 153         | January    | -1954           |
| 153         | February   | -776            |
| 153         | March      | -1695           |
| 154         | January    | -1392           |
| 154         | February   | -2340           |
| 154         | March      | -2104           |
| 154         | April      | -2555           |
| 155         | January    | -996            |
| 155         | February   | -2941           |
| 155         | March      | -3377           |
| 155         | April      | -4530           |
| 156         | January    | 82              |
| 156         | April      | 312             |
| 157         | January    | 138             |
| 157         | February   | -611            |
| 157         | March      | 2766            |
| 158         | January    | 56              |
| 158         | February   | -136            |
| 158         | March      | -1106           |
| 159         | January    | -301            |
| 160         | January    | 843             |
| 160         | February   | 543             |
| 160         | March      | -69             |
| 160         | April      | -307            |
| 161         | January    | -1121           |
| 161         | February   | -961            |
| 161         | March      | -291            |
| 162         | January    | 123             |
| 162         | February   | 784             |
| 163         | January    | -73             |
| 163         | February   | -328            |
| 163         | March      | -3116           |
| 163         | April      | -3055           |
| 164         | January    | 548             |
| 164         | February   | 957             |
| 165         | January    | -61             |
| 165         | February   | -1088           |
| 165         | March      | -3701           |
| 165         | April      | -3931           |
| 166         | January    | 957             |
| 166         | February   | 1546            |
| 166         | March      | 1303            |
| 166         | April      | 1783            |
| 167         | January    | 51              |
| 167         | February   | 574             |
| 167         | March      | -566            |
| 167         | April      | -748            |
| 168         | January    | 114             |
| 168         | February   | -801            |
| 169         | January    | -569            |
| 169         | February   | -1190           |
| 169         | March      | 9               |
| 169         | April      | 906             |
| 170         | January    | -38             |
| 170         | February   | -373            |
| 170         | March      | -137            |
| 170         | April      | -850            |
| 171         | January    | -197            |
| 171         | February   | -1400           |
| 171         | March      | -1921           |
| 171         | April      | -911            |
| 172         | January    | -174            |
| 172         | March      | -1038           |
| 173         | January    | 1298            |
| 173         | February   | 1398            |
| 173         | March      | 912             |
| 173         | April      | 121             |
| 174         | January    | 1142            |
| 174         | February   | -98             |
| 174         | March      | -1135           |
| 174         | April      | 644             |
| 175         | January    | -326            |
| 175         | February   | -755            |
| 175         | March      | -1822           |
| 175         | April      | -1549           |
| 176         | January    | 655             |
| 176         | February   | 605             |
| 176         | March      | -531            |
| 176         | April      | -1067           |
| 177         | January    | 405             |
| 177         | February   | -156            |
| 177         | March      | 800             |
| 177         | April      | -974            |
| 178         | January    | 252             |
| 178         | February   | 387             |
| 178         | March      | 390             |
| 178         | April      | -1983           |
| 179         | January    | -1754           |
| 179         | February   | -3386           |
| 179         | March      | -7953           |
| 180         | January    | -838            |
| 180         | February   | -1808           |
| 180         | March      | -2804           |
| 180         | April      | -3175           |
| 181         | January    | -47             |
| 181         | February   | -843            |
| 181         | March      | -2640           |
| 182         | January    | 97              |
| 182         | February   | -45             |
| 182         | March      | -843            |
| 182         | April      | -159            |
| 183         | January    | -540            |
| 183         | February   | -3729           |
| 183         | March      | -6083           |
| 183         | April      | -6560           |
| 184         | January    | 472             |
| 184         | February   | -331            |
| 184         | March      | -2530           |
| 184         | April      | -3178           |
| 185         | January    | 626             |
| 185         | February   | -11             |
| 185         | March      | -1001           |
| 185         | April      | -505            |
| 186         | January    | 534             |
| 186         | February   | 1345            |
| 186         | March      | 1930            |
| 186         | April      | 1284            |
| 187         | January    | -211            |
| 187         | February   | -1379           |
| 187         | March      | -3060           |
| 187         | April      | -2272           |
| 188         | January    | -184            |
| 188         | February   | 1013            |
| 188         | March      | 52              |
| 188         | April      | -475            |
| 189         | January    | -838            |
| 189         | February   | -2101           |
| 189         | March      | -4007           |
| 190         | January    | 14              |
| 190         | February   | 459             |
| 190         | March      | 523             |
| 190         | April      | 1178            |
| 191         | January    | 1632            |
| 191         | February   | 1306            |
| 191         | March      | 1036            |
| 191         | April      | 879             |
| 192         | January    | 2526            |
| 192         | February   | -689            |
| 192         | March      | 383             |
| 192         | April      | 1139            |
| 193         | January    | 689             |
| 193         | March      | 486             |
| 194         | January    | 137             |
| 194         | February   | -2211           |
| 194         | March      | 178             |
| 194         | April      | -697            |
| 195         | January    | 489             |
| 195         | March      | 406             |
| 196         | January    | 734             |
| 196         | February   | 1295            |
| 196         | March      | 1382            |
| 197         | January    | -446            |
| 197         | February   | 137             |
| 197         | March      | 1023            |
| 197         | April      | 3685            |
| 198         | January    | 1144            |
| 198         | February   | 288             |
| 198         | March      | -1253           |
| 198         | April      | -757            |
| 199         | January    | 530             |
| 199         | February   | 515             |
| 199         | March      | -14             |
| 199         | April      | -220            |
| 200         | January    | 997             |
| 200         | February   | 1356            |
| 200         | March      | 2914            |
| 200         | April      | 2853            |
| 201         | January    | -383            |
| 201         | February   | -292            |
| 201         | March      | 1529            |
| 202         | January    | -530            |
| 202         | February   | -916            |
| 202         | March      | -2415           |
| 203         | January    | 2528            |
| 203         | February   | 3471            |
| 203         | March      | 3461            |
| 203         | April      | 3437            |
| 204         | January    | 749             |
| 204         | February   | 1039            |
| 204         | March      | 1587            |
| 204         | April      | 1893            |
| 205         | January    | -82             |
| 205         | February   | 1211            |
| 205         | March      | 1067            |
| 206         | January    | -215            |
| 206         | February   | -949            |
| 206         | March      | -4974           |
| 206         | April      | -5374           |
| 207         | January    | 322             |
| 207         | February   | -1954           |
| 207         | March      | -2152           |
| 207         | April      | -1014           |
| 208         | January    | 537             |
| 208         | February   | 406             |
| 208         | April      | 1361            |
| 209         | January    | -202            |
| 209         | February   | -766            |
| 209         | March      | -1266           |
| 209         | April      | -2351           |
| 210         | January    | 60              |
| 210         | February   | -1361           |
| 210         | March      | -1309           |
| 210         | April      | -792            |
| 211         | January    | 607             |
| 211         | February   | 1839            |
| 211         | March      | 257             |
| 211         | April      | -602            |
| 212         | January    | -336            |
| 212         | February   | 481             |
| 212         | March      | 3529            |
| 213         | January    | -239            |
| 213         | February   | -1199           |
| 213         | March      | -1184           |
| 213         | April      | -1717           |
| 214         | January    | -445            |
| 214         | February   | -1511           |
| 214         | March      | 83              |
| 214         | April      | 802             |
| 215         | January    | 822             |
| 215         | February   | 1770            |
| 215         | March      | 697             |
| 215         | April      | 414             |
| 216         | January    | 1619            |
| 216         | February   | 3302            |
| 216         | March      | 872             |
| 216         | April      | -110            |
| 217         | January    | 870             |
| 217         | February   | 1839            |
| 217         | March      | 9               |
| 218         | January    | 208             |
| 218         | February   | -1620           |
| 218         | March      | -465            |
| 218         | April      | 1167            |
| 219         | January    | 165             |
| 219         | February   | -845            |
| 219         | March      | 1108            |
| 219         | April      | 306             |
| 220         | January    | 307             |
| 220         | February   | 714             |
| 220         | March      | -29             |
| 220         | April      | -958            |
| 221         | January    | 1384            |
| 221         | February   | 1481            |
| 221         | March      | 1055            |
| 222         | January    | 657             |
| 222         | February   | 1997            |
| 222         | March      | 1136            |
| 222         | April      | 1532            |
| 223         | January    | 396             |
| 223         | February   | -1100           |
| 223         | March      | -1723           |
| 223         | April      | -2435           |
| 224         | January    | 487             |
| 224         | February   | -206            |
| 224         | March      | -1581           |
| 224         | April      | -1369           |
| 225         | January    | 280             |
| 225         | February   | -89             |
| 225         | March      | 297             |
| 226         | January    | -980            |
| 226         | February   | -586            |
| 226         | March      | -1855           |
| 226         | April      | -1430           |
| 227         | January    | -622            |
| 227         | February   | -1045           |
| 227         | March      | -2336           |
| 227         | April      | -3161           |
| 228         | January    | 294             |
| 228         | February   | -253            |
| 228         | March      | -1724           |
| 229         | January    | 621             |
| 229         | February   | 1618            |
| 229         | March      | 703             |
| 230         | January    | 499             |
| 230         | February   | 990             |
| 230         | March      | 2428            |
| 230         | April      | 2356            |
| 231         | January    | -236            |
| 231         | February   | -534            |
| 231         | March      | -2010           |
| 231         | April      | -2475           |
| 232         | January    | 1418            |
| 232         | February   | 864             |
| 232         | March      | 923             |
| 233         | January    | 1795            |
| 233         | February   | 2910            |
| 233         | March      | 3742            |
| 234         | January    | -200            |
| 234         | February   | 322             |
| 234         | March      | -2276           |
| 235         | January    | -1963           |
| 235         | February   | -476            |
| 235         | March      | 24              |
| 236         | January    | 356             |
| 236         | February   | 1059            |
| 236         | March      | 924             |
| 236         | April      | -100            |
| 237         | January    | -174            |
| 237         | February   | 136             |
| 237         | March      | 1567            |
| 237         | April      | 812             |
| 238         | January    | 802             |
| 238         | February   | 1270            |
| 238         | April      | -479            |
| 239         | January    | -10             |
| 239         | February   | 706             |
| 239         | March      | 574             |
| 239         | April      | 1871            |
| 240         | January    | 1108            |
| 240         | February   | 754             |
| 240         | March      | 3200            |
| 240         | April      | 3075            |
| 241         | January    | 20              |
| 242         | January    | 1143            |
| 242         | February   | -462            |
| 242         | March      | -1891           |
| 242         | April      | -3334           |
| 243         | January    | -368            |
| 243         | March      | 979             |
| 244         | January    | 728             |
| 244         | February   | 1752            |
| 244         | March      | 1930            |
| 244         | April      | 877             |
| 245         | January    | 76              |
| 245         | February   | -341            |
| 245         | March      | -1791           |
| 245         | April      | -1021           |
| 246         | January    | 506             |
| 246         | February   | 584             |
| 246         | March      | 277             |
| 246         | April      | -2132           |
| 247         | January    | 983             |
| 247         | February   | 577             |
| 247         | March      | 297             |
| 248         | January    | 304             |
| 248         | February   | 667             |
| 248         | March      | 955             |
| 248         | April      | 1188            |
| 249         | January    | 336             |
| 249         | March      | 1065            |
| 249         | April      | 895             |
| 250         | January    | 149             |
| 250         | February   | 0               |
| 250         | March      | 177             |
| 250         | April      | -1555           |
| 251         | January    | 1276            |
| 251         | February   | -924            |
| 251         | March      | -1230           |
| 251         | April      | -1883           |
| 252         | January    | 289             |
| 252         | April      | 133             |
| 253         | January    | -578            |
| 253         | February   | -577            |
| 253         | March      | -457            |
| 253         | April      | -484            |
| 254         | January    | 36              |
| 254         | February   | -2883           |
| 254         | March      | -590            |
| 255         | January    | 253             |
| 255         | February   | 129             |
| 255         | March      | -548            |
| 256         | January    | 1743            |
| 256         | February   | 906             |
| 256         | March      | 1152            |
| 256         | April      | 1094            |
| 257         | January    | 414             |
| 257         | February   | -1609           |
| 257         | March      | -472            |
| 257         | April      | -976            |
| 258         | January    | 590             |
| 258         | February   | -1076           |
| 258         | March      | -2893           |
| 258         | April      | -1465           |
| 259         | January    | 928             |
| 259         | February   | -267            |
| 259         | March      | -1458           |
| 260         | January    | 1865            |
| 260         | March      | 1909            |
| 261         | January    | 746             |
| 261         | February   | 1408            |
| 261         | March      | -329            |
| 261         | April      | -31             |
| 262         | January    | -1070           |
| 262         | February   | -2599           |
| 262         | March      | -2372           |
| 263         | January    | 312             |
| 263         | February   | 112             |
| 263         | April      | 770             |
| 264         | January    | 770             |
| 264         | February   | 1545            |
| 264         | March      | 2088            |
| 264         | April      | 1295            |
| 265         | January    | -25             |
| 265         | February   | -1481           |
| 265         | March      | -2592           |
| 265         | April      | -1948           |
| 266         | January    | 651             |
| 266         | February   | 1455            |
| 266         | March      | 787             |
| 266         | April      | 1138            |
| 267         | January    | -193            |
| 267         | February   | -2068           |
| 267         | March      | -5236           |
| 267         | April      | -2442           |
| 268         | January    | 1699            |
| 268         | February   | 123             |
| 268         | March      | 270             |
| 268         | April      | -218            |
| 269         | January    | -2665           |
| 269         | February   | -3985           |
| 269         | March      | -2470           |
| 269         | April      | -1864           |
| 270         | January    | 1395            |
| 270         | February   | 961             |
| 270         | March      | 599             |
| 270         | April      | -442            |
| 271         | January    | -1586           |
| 271         | February   | 272             |
| 271         | March      | -616            |
| 271         | April      | 180             |
| 272         | January    | -228            |
| 272         | February   | -1673           |
| 272         | April      | -1769           |
| 273         | January    | 876             |
| 273         | February   | 133             |
| 273         | March      | -178            |
| 273         | April      | 308             |
| 274         | January    | -780            |
| 274         | February   | -582            |
| 274         | March      | 124             |
| 275         | January    | 211             |
| 275         | February   | -1325           |
| 275         | March      | -2973           |
| 275         | April      | -3169           |
| 276         | January    | -851            |
| 276         | February   | -796            |
| 276         | March      | -1944           |
| 277         | January    | 615             |
| 277         | February   | 1411            |
| 277         | March      | 1865            |
| 278         | January    | 1309            |
| 278         | February   | 1723            |
| 278         | March      | 2026            |
| 278         | April      | 3554            |
| 279         | January    | 1895            |
| 279         | February   | 3641            |
| 279         | March      | 4183            |
| 279         | April      | 4103            |
| 280         | January    | -87             |
| 280         | February   | -185            |
| 280         | March      | -373            |
| 281         | January    | 220             |
| 281         | February   | 1055            |
| 281         | March      | 2192            |
| 281         | April      | 3004            |
| 282         | January    | 74              |
| 282         | February   | -713            |
| 282         | March      | -1661           |
| 282         | April      | -1311           |
| 283         | January    | -1201           |
| 283         | February   | -2818           |
| 283         | March      | -5721           |
| 283         | April      | -7145           |
| 284         | January    | 257             |
| 284         | February   | -2602           |
| 284         | March      | -2826           |
| 284         | April      | -3379           |
| 285         | January    | 360             |
| 285         | February   | 1358            |
| 285         | March      | 1965            |
| 286         | January    | 177             |
| 286         | February   | 171             |
| 287         | January    | 658             |
| 287         | February   | 829             |
| 287         | March      | 886             |
| 287         | April      | -406            |
| 288         | January    | 778             |
| 288         | February   | -867            |
| 288         | March      | -515            |
| 289         | January    | 838             |
| 289         | February   | -207            |
| 289         | March      | -934            |
| 289         | April      | -1059           |
| 290         | January    | 785             |
| 290         | February   | 1139            |
| 290         | March      | 2061            |
| 290         | April      | 21              |
| 291         | January    | 930             |
| 291         | April      | 531             |
| 292         | January    | -3458           |
| 292         | February   | -4646           |
| 292         | March      | -4760           |
| 293         | January    | -383            |
| 293         | February   | -1452           |
| 293         | March      | -1770           |
| 293         | April      | -2500           |
| 294         | January    | 307             |
| 294         | February   | 1557            |
| 294         | March      | 707             |
| 295         | January    | 636             |
| 295         | February   | 496             |
| 295         | March      | 1430            |
| 295         | April      | -177            |
| 296         | January    | 191             |
| 296         | February   | 1152            |
| 296         | March      | 1309            |
| 296         | April      | 2220            |
| 297         | January    | 550             |
| 297         | February   | 585             |
| 297         | March      | 1004            |
| 297         | April      | 1282            |
| 298         | January    | 278             |
| 298         | February   | -580            |
| 298         | March      | 767             |
| 298         | April      | 1489            |
| 299         | January    | 961             |
| 299         | February   | 1246            |
| 299         | March      | 70              |
| 300         | January    | 672             |
| 300         | February   | -949            |
| 300         | March      | -2374           |
| 300         | April      | -3179           |
| 301         | January    | -906            |
| 301         | February   | -1977           |
| 301         | March      | -3636           |
| 301         | April      | -3529           |
| 302         | January    | -1499           |
| 302         | February   | -2077           |
| 302         | March      | -2410           |
| 302         | April      | -1795           |
| 303         | January    | 332             |
| 303         | February   | 465             |
| 303         | March      | -629            |
| 303         | April      | -660            |
| 304         | January    | 152             |
| 304         | February   | -1360           |
| 304         | March      | -2420           |
| 304         | April      | -2047           |
| 305         | January    | 20              |
| 305         | February   | 189             |
| 305         | March      | -56             |
| 306         | January    | 402             |
| 306         | March      | 0               |
| 306         | April      | 1565            |
| 307         | January    | -696            |
| 307         | February   | 750             |
| 307         | March      | 525             |
| 307         | April      | 62              |
| 308         | January    | -561            |
| 308         | February   | 316             |
| 308         | March      | 710             |
| 308         | April      | 971             |
| 309         | January    | -363            |
| 309         | February   | -2404           |
| 309         | March      | -1813           |
| 309         | April      | -960            |
| 310         | January    | 860             |
| 310         | February   | 156             |
| 310         | March      | 3066            |
| 311         | January    | 310             |
| 311         | February   | 1006            |
| 311         | March      | -955            |
| 311         | April      | -1055           |
| 312         | January    | 485             |
| 312         | February   | 656             |
| 312         | March      | -1065           |
| 312         | April      | -2318           |
| 313         | January    | 901             |
| 313         | February   | 972             |
| 313         | March      | -1311           |
| 313         | April      | -34             |
| 314         | January    | 448             |
| 314         | February   | -633            |
| 314         | March      | 91              |
| 314         | April      | -249            |
| 315         | January    | 1295            |
| 315         | February   | 1349            |
| 315         | March      | 2287            |
| 316         | January    | 184             |
| 316         | February   | -2483           |
| 316         | March      | -3299           |
| 317         | January    | 869             |
| 317         | February   | 1232            |
| 317         | April      | 995             |
| 318         | January    | 321             |
| 318         | February   | -342            |
| 318         | March      | -1648           |
| 319         | January    | 83              |
| 319         | February   | -703            |
| 319         | March      | -488            |
| 320         | January    | 2426            |
| 320         | February   | 1909            |
| 320         | March      | 2239            |
| 320         | April      | 2851            |
| 321         | January    | 243             |
| 321         | February   | -213            |
| 321         | April      | 572             |
| 322         | January    | 1949            |
| 322         | February   | 2471            |
| 322         | March      | 1811            |
| 323         | January    | 1323            |
| 323         | February   | -1880           |
| 323         | March      | -4196           |
| 323         | April      | -5221           |
| 324         | January    | 203             |
| 324         | February   | 967             |
| 324         | March      | 1470            |
| 325         | January    | 60              |
| 325         | February   | -1878           |
| 325         | March      | -1862           |
| 326         | January    | -211            |
| 326         | February   | 417             |
| 327         | January    | 919             |
| 327         | March      | -164            |
| 328         | January    | -1232           |
| 328         | February   | -3426           |
| 328         | March      | -4703           |
| 328         | April      | -4559           |
| 329         | January    | 831             |
| 329         | February   | 2               |
| 329         | March      | -626            |
| 329         | April      | 736             |
| 330         | January    | 826             |
| 330         | February   | 1099            |
| 330         | March      | -666            |
| 330         | April      | -1474           |
| 331         | January    | -54             |
| 331         | February   | -173            |
| 331         | March      | -949            |
| 332         | January    | 202             |
| 332         | February   | 559             |
| 332         | March      | 494             |
| 332         | April      | 670             |
| 333         | January    | -229            |
| 333         | February   | -127            |
| 333         | March      | 567             |
| 333         | April      | 920             |
| 334         | January    | 1177            |
| 334         | February   | 2724            |
| 334         | March      | 2425            |
| 334         | April      | 1614            |
| 335         | January    | 570             |
| 335         | February   | -354            |
| 335         | March      | 423             |
| 336         | January    | 543             |
| 336         | February   | -595            |
| 336         | March      | 135             |
| 336         | April      | 599             |
| 337         | January    | -264            |
| 337         | February   | 170             |
| 337         | March      | 1450            |
| 337         | April      | 1314            |
| 338         | January    | 262             |
| 338         | February   | 533             |
| 338         | March      | 2767            |
| 338         | April      | 1264            |
| 339         | January    | -780            |
| 339         | February   | 1088            |
| 339         | March      | 625             |
| 340         | January    | -1086           |
| 340         | February   | 276             |
| 340         | March      | 559             |
| 340         | April      | 1390            |
| 341         | January    | 345             |
| 341         | February   | -873            |
| 341         | March      | -2133           |
| 341         | April      | -2094           |
| 342         | January    | 347             |
| 342         | February   | -285            |
| 342         | March      | 503             |
| 342         | April      | -135            |
| 343         | January    | 1339            |
| 343         | February   | 1653            |
| 343         | March      | 1841            |
| 344         | January    | -932            |
| 344         | February   | 505             |
| 344         | March      | 1475            |
| 345         | January    | -100            |
| 345         | February   | -650            |
| 345         | March      | -2288           |
| 346         | January    | 916             |
| 346         | February   | -1052           |
| 346         | April      | -3802           |
| 347         | January    | 394             |
| 347         | February   | -775            |
| 347         | March      | -1768           |
| 348         | January    | -771            |
| 348         | February   | 114             |
| 348         | March      | -155            |
| 348         | April      | 48              |
| 349         | January    | -844            |
| 349         | February   | -1040           |
| 349         | March      | 1309            |
| 349         | April      | 1964            |
| 350         | January    | 2200            |
| 350         | February   | 1080            |
| 350         | March      | -174            |
| 350         | April      | -1233           |
| 351         | January    | 90              |
| 351         | February   | -1533           |
| 351         | March      | -1860           |
| 352         | January    | 416             |
| 352         | February   | -1612           |
| 352         | March      | -1774           |
| 352         | April      | -2269           |
| 353         | January    | -555            |
| 353         | February   | -1819           |
| 353         | March      | -516            |
| 354         | January    | 822             |
| 354         | March      | 158             |
| 355         | January    | -245            |
| 355         | February   | -194            |
| 355         | March      | -1137           |
| 355         | April      | -852            |
| 356         | January    | -1870           |
| 356         | February   | -3929           |
| 356         | March      | -6518           |
| 356         | April      | -6882           |
| 357         | January    | 780             |
| 357         | February   | 878             |
| 357         | March      | 496             |
| 357         | April      | -188            |
| 358         | January    | -1062           |
| 358         | February   | -619            |
| 358         | March      | -883            |
| 358         | April      | -708            |
| 359         | January    | 890             |
| 359         | February   | 1284            |
| 359         | March      | 3159            |
| 359         | April      | 2878            |
| 360         | January    | -1306           |
| 360         | February   | -427            |
| 360         | March      | -1255           |
| 360         | April      | -324            |
| 361         | January    | 340             |
| 361         | February   | 772             |
| 362         | January    | 416             |
| 362         | February   | 481             |
| 362         | March      | -564            |
| 362         | April      | 615             |
| 363         | January    | 977             |
| 363         | February   | -618            |
| 363         | March      | -3065           |
| 363         | April      | -2886           |
| 364         | January    | -57             |
| 364         | February   | -456            |
| 364         | March      | 1173            |
| 364         | April      | 811             |
| 365         | January    | -68             |
| 365         | February   | 244             |
| 365         | March      | -75             |
| 365         | April      | -760            |
| 366         | January    | -51             |
| 366         | February   | -93             |
| 366         | March      | -1305           |
| 366         | April      | -1096           |
| 367         | January    | 239             |
| 367         | February   | -467            |
| 367         | March      | -3101           |
| 367         | April      | -1697           |
| 368         | January    | -526            |
| 368         | February   | -3490           |
| 368         | March      | -1476           |
| 368         | April      | -2860           |
| 369         | January    | 266             |
| 369         | March      | 1679            |
| 370         | January    | -2295           |
| 370         | February   | -1635           |
| 370         | March      | -1311           |
| 370         | April      | -1496           |
| 371         | January    | -134            |
| 371         | February   | -154            |
| 371         | March      | -5              |
| 371         | April      | -1243           |
| 372         | January    | 2718            |
| 372         | February   | 1241            |
| 372         | March      | -1625           |
| 373         | January    | 493             |
| 373         | February   | 277             |
| 373         | March      | 1057            |
| 373         | April      | 1451            |
| 374         | January    | -457            |
| 374         | February   | -1292           |
| 374         | March      | -846            |
| 375         | January    | 647             |
| 375         | February   | 328             |
| 375         | March      | -440            |
| 375         | April      | -1291           |
| 376         | January    | 1614            |
| 376         | February   | 2515            |
| 376         | March      | 3062            |
| 377         | January    | 252             |
| 377         | February   | -182            |
| 377         | April      | -785            |
| 378         | January    | 484             |
| 378         | February   | 2424            |
| 378         | March      | 1590            |
| 379         | January    | -35             |
| 379         | February   | -268            |
| 379         | April      | -1206           |
| 380         | January    | -849            |
| 380         | February   | -1481           |
| 380         | March      | -3662           |
| 381         | January    | 66              |
| 381         | February   | 992             |
| 381         | March      | 278             |
| 381         | April      | -597            |
| 382         | January    | -687            |
| 382         | February   | -1195           |
| 382         | March      | -1141           |
| 383         | January    | -36             |
| 383         | February   | 935             |
| 383         | March      | -617            |
| 383         | April      | 913             |
| 384         | January    | -10             |
| 384         | February   | -2486           |
| 384         | March      | -2527           |
| 385         | January    | -1174           |
| 385         | March      | -4693           |
| 385         | April      | -4861           |
| 386         | January    | 1108            |
| 386         | February   | 759             |
| 386         | March      | -766            |
| 386         | April      | -3837           |
| 387         | January    | 1069            |
| 387         | March      | 2551            |
| 387         | April      | 2454            |
| 388         | January    | 2243            |
| 388         | February   | 1126            |
| 388         | March      | 1598            |
| 388         | April      | 1376            |
| 389         | January    | -27             |
| 389         | February   | 490             |
| 389         | March      | 1214            |
| 389         | April      | 2005            |
| 390         | January    | -705            |
| 390         | February   | -2038           |
| 390         | March      | -1929           |
| 390         | April      | -2801           |
| 391         | January    | 603             |
| 391         | February   | 2               |
| 391         | March      | 272             |
| 391         | April      | -90             |
| 392         | January    | 816             |
| 392         | February   | 2034            |
| 392         | March      | 1498            |
| 392         | April      | 1253            |
| 393         | January    | 659             |
| 393         | February   | 118             |
| 393         | March      | 1500            |
| 393         | April      | 639             |
| 394         | January    | 3268            |
| 394         | February   | 2644            |
| 394         | March      | 1420            |
| 395         | January    | -1782           |
| 395         | February   | -914            |
| 395         | March      | -903            |
| 395         | April      | -1723           |
| 396         | January    | -909            |
| 396         | February   | -550            |
| 396         | March      | -2848           |
| 397         | January    | 973             |
| 397         | February   | 1106            |
| 397         | March      | 1709            |
| 398         | January    | -429            |
| 398         | February   | -3171           |
| 398         | March      | -3401           |
| 399         | January    | 593             |
| 399         | February   | -301            |
| 399         | March      | -1488           |
| 399         | April      | -1717           |
| 400         | January    | 155             |
| 400         | February   | -409            |
| 400         | April      | 1338            |
| 401         | January    | 102             |
| 401         | February   | -25             |
| 401         | March      | 83              |
| 402         | January    | 1478            |
| 402         | February   | 1599            |
| 402         | March      | 796             |
| 403         | January    | 303             |
| 403         | March      | 987             |
| 403         | April      | 1047            |
| 404         | January    | -245            |
| 404         | February   | -347            |
| 404         | March      | -2484           |
| 405         | January    | -2897           |
| 405         | February   | -4125           |
| 405         | March      | -6882           |
| 405         | April      | -7070           |
| 406         | January    | 795             |
| 406         | February   | 1131            |
| 406         | March      | 2597            |
| 406         | April      | 2279            |
| 407         | January    | 7               |
| 407         | February   | 46              |
| 407         | March      | -900            |
| 407         | April      | -3275           |
| 408         | January    | -145            |
| 408         | March      | 800             |
| 408         | April      | -132            |
| 409         | January    | 155             |
| 409         | February   | 1371            |
| 409         | March      | 2455            |
| 409         | April      | 2623            |
| 410         | January    | 1025            |
| 410         | February   | 199             |
| 410         | March      | -52             |
| 411         | January    | 551             |
| 411         | April      | -981            |
| 412         | January    | 722             |
| 412         | February   | 608             |
| 413         | January    | 642             |
| 413         | February   | 1072            |
| 413         | April      | 801             |
| 414         | January    | 439             |
| 414         | March      | 1918            |
| 415         | January    | 331             |
| 415         | February   | -586            |
| 415         | March      | -4287           |
| 416         | January    | 756             |
| 416         | February   | 1715            |
| 416         | March      | 3324            |
| 416         | April      | 3898            |
| 417         | January    | 707             |
| 417         | February   | -1079           |
| 417         | March      | -1540           |
| 417         | April      | -1847           |
| 418         | January    | -499            |
| 418         | February   | -996            |
| 418         | March      | -1624           |
| 418         | April      | -1828           |
| 419         | January    | 1193            |
| 419         | February   | 123             |
| 419         | March      | -1280           |
| 420         | January    | -280            |
| 420         | February   | -2117           |
| 420         | March      | -2457           |
| 420         | April      | -2078           |
| 421         | January    | -741            |
| 421         | February   | -571            |
| 421         | March      | -413            |
| 421         | April      | 312             |
| 422         | January    | 356             |
| 422         | February   | -1661           |
| 422         | March      | -1046           |
| 422         | April      | -3357           |
| 423         | January    | 361             |
| 423         | February   | -262            |
| 423         | March      | 1321            |
| 424         | January    | -595            |
| 424         | February   | -648            |
| 424         | March      | -940            |
| 424         | April      | -314            |
| 425         | January    | 63              |
| 425         | February   | -505            |
| 425         | March      | 57              |
| 425         | April      | -721            |
| 426         | January    | -880            |
| 426         | February   | -3802           |
| 426         | March      | -4352           |
| 426         | April      | -5352           |
| 427         | January    | 588             |
| 427         | February   | 1305            |
| 427         | March      | 676             |
| 427         | April      | -316            |
| 428         | January    | 280             |
| 428         | February   | 687             |
| 428         | March      | 1217            |
| 429         | January    | 82              |
| 429         | February   | 473             |
| 429         | March      | -46             |
| 429         | April      | -901            |
| 430         | January    | -8              |
| 430         | February   | 403             |
| 430         | March      | -1326           |
| 431         | January    | -400            |
| 431         | February   | -1139           |
| 432         | January    | 392             |
| 432         | February   | 986             |
| 432         | March      | 963             |
| 432         | April      | 2435            |
| 433         | January    | 883             |
| 433         | February   | 1286            |
| 433         | March      | 660             |
| 434         | January    | 1123            |
| 434         | February   | -117            |
| 434         | March      | -2366           |
| 434         | April      | -1815           |
| 435         | January    | -1329           |
| 435         | February   | -38             |
| 435         | March      | -1176           |
| 436         | January    | 917             |
| 436         | February   | 886             |
| 436         | March      | -676            |
| 437         | January    | -361            |
| 437         | February   | -1537           |
| 437         | March      | -1318           |
| 437         | April      | -1134           |
| 438         | January    | 1317            |
| 438         | February   | 2813            |
| 438         | March      | 1423            |
| 439         | January    | 430             |
| 439         | March      | -381            |
| 439         | April      | 318             |
| 440         | January    | -123            |
| 440         | February   | 146             |
| 440         | March      | 490             |
| 441         | January    | -329            |
| 441         | February   | 745             |
| 441         | March      | -669            |
| 441         | April      | -798            |
| 442         | January    | 142             |
| 442         | February   | -2898           |
| 442         | March      | -4520           |
| 442         | April      | -5499           |
| 443         | January    | 760             |
| 443         | February   | 710             |
| 443         | March      | -359            |
| 443         | April      | -11             |
| 444         | January    | 83              |
| 444         | February   | -1178           |
| 444         | March      | -499            |
| 444         | April      | -820            |
| 445         | January    | 1364            |
| 445         | February   | 894             |
| 445         | March      | 724             |
| 445         | April      | 312             |
| 446         | January    | 412             |
| 446         | March      | -53             |
| 446         | April      | 405             |
| 447         | January    | 1195            |
| 447         | February   | 41              |
| 447         | March      | -1290           |
| 448         | January    | 1360            |
| 448         | February   | -909            |
| 448         | March      | -2042           |
| 448         | April      | -2418           |
| 449         | January    | -3100           |
| 449         | February   | -3928           |
| 449         | March      | -3045           |
| 450         | January    | 469             |
| 450         | February   | -159            |
| 450         | March      | -737            |
| 450         | April      | -36             |
| 451         | January    | 910             |
| 451         | February   | -1313           |
| 451         | March      | -645            |
| 451         | April      | -756            |
| 452         | January    | 1360            |
| 452         | February   | 1654            |
| 452         | March      | 1613            |
| 453         | January    | 638             |
| 453         | February   | 811             |
| 453         | March      | -595            |
| 453         | April      | 117             |
| 454         | January    | 11              |
| 454         | February   | 2163            |
| 454         | March      | 2101            |
| 455         | January    | 329             |
| 455         | March      | -231            |
| 456         | January    | 1314            |
| 456         | February   | 744             |
| 456         | March      | -55             |
| 456         | April      | 12              |
| 457         | January    | 195             |
| 457         | February   | -234            |
| 457         | March      | -714            |
| 457         | April      | -718            |
| 458         | January    | 715             |
| 458         | February   | -653            |
| 459         | January    | 246             |
| 459         | February   | -2912           |
| 459         | March      | -2834           |
| 460         | January    | 80              |
| 460         | February   | -1158           |
| 460         | March      | -1175           |
| 460         | April      | -327            |
| 461         | January    | 2267            |
| 461         | February   | 3431            |
| 461         | March      | 3212            |
| 462         | January    | 907             |
| 462         | February   | -10             |
| 462         | March      | -831            |
| 462         | April      | -1395           |
| 463         | January    | 1166            |
| 463         | February   | 312             |
| 463         | March      | 673             |
| 463         | April      | 280             |
| 464         | January    | 953             |
| 464         | March      | -511            |
| 464         | April      | -1494           |
| 465         | January    | 955             |
| 465         | February   | 1989            |
| 465         | March      | 1506            |
| 465         | April      | 1350            |
| 466         | January    | 80              |
| 466         | February   | -1979           |
| 466         | March      | -2113           |
| 467         | January    | 1994            |
| 467         | February   | 3582            |
| 467         | March      | 2754            |
| 467         | April      | 1190            |
| 468         | January    | 39              |
| 468         | February   | -155            |
| 468         | March      | -917            |
| 469         | January    | 386             |
| 469         | February   | 2161            |
| 469         | March      | -802            |
| 469         | April      | -1537           |
| 470         | January    | 377             |
| 470         | February   | -311            |
| 471         | January    | 781             |
| 471         | March      | 1238            |
| 471         | April      | 1887            |
| 472         | January    | 811             |
| 472         | February   | -115            |
| 472         | March      | 32              |
| 472         | April      | 218             |
| 473         | January    | -183            |
| 473         | February   | -864            |
| 473         | March      | -3630           |
| 473         | April      | -5772           |
| 474         | January    | 928             |
| 474         | February   | 139             |
| 474         | March      | -259            |
| 475         | January    | -673            |
| 475         | February   | -1966           |
| 475         | March      | -5073           |
| 476         | January    | -476            |
| 476         | February   | -2003           |
| 476         | March      | -3365           |
| 476         | April      | -4972           |
| 477         | January    | -3034           |
| 477         | February   | -4592           |
| 477         | March      | -6538           |
| 478         | January    | -712            |
| 478         | February   | 2278            |
| 478         | March      | 2087            |
| 479         | January    | 320             |
| 479         | February   | -327            |
| 479         | March      | 513             |
| 480         | January    | 522             |
| 480         | March      | -235            |
| 480         | April      | -165            |
| 481         | January    | -1396           |
| 481         | February   | -2905           |
| 481         | March      | -3394           |
| 482         | January    | 386             |
| 482         | February   | -687            |
| 482         | March      | -1256           |
| 483         | January    | 2038            |
| 483         | March      | -189            |
| 483         | April      | 1330            |
| 484         | January    | 871             |
| 484         | March      | 1796            |
| 485         | January    | 16              |
| 485         | February   | 1507            |
| 485         | March      | 2202            |
| 486         | January    | -1632           |
| 486         | February   | -2250           |
| 486         | March      | -3108           |
| 487         | January    | -572            |
| 487         | February   | 312             |
| 487         | March      | 162             |
| 487         | April      | -330            |
| 488         | January    | -243            |
| 488         | February   | 297             |
| 488         | March      | -412            |
| 488         | April      | -191            |
| 489         | January    | 556             |
| 489         | February   | 1808            |
| 489         | March      | 3342            |
| 489         | April      | 5338            |
| 490         | January    | 271             |
| 490         | February   | 342             |
| 490         | April      | 24              |
| 491         | January    | -3              |
| 491         | February   | 298             |
| 491         | March      | -2319           |
| 492         | January    | -738            |
| 492         | February   | -1399           |
| 492         | March      | -2133           |
| 493         | January    | 845             |
| 493         | February   | -824            |
| 493         | April      | -738            |
| 494         | January    | 529             |
| 494         | February   | 909             |
| 494         | March      | 1447            |
| 495         | January    | -286            |
| 495         | February   | -1438           |
| 495         | March      | -89             |
| 496         | January    | 47              |
| 496         | February   | -3076           |
| 496         | March      | -2426           |
| 497         | January    | 754             |
| 497         | February   | 1003            |
| 497         | March      | 1739            |
| 497         | April      | 2680            |
| 498         | January    | 1360            |
| 498         | February   | 2195            |
| 498         | March      | 2989            |
| 498         | April      | 3488            |
| 499         | January    | -304            |
| 499         | February   | 1415            |
| 499         | March      | 599             |
| 500         | January    | 1594            |
| 500         | February   | 2981            |
| 500         | March      | 2251            |

---
**Query #5** What is the percentage of customers who increase their closing balance by more than 5%?

    WITH txn_cte AS
    (SELECT customer_id, EXTRACT(MONTH FROM txn_date) as month_num, TO_CHAR(txn_date, 'Month') as month_name,
     SUM(CASE
         	WHEN txn_type = 'deposit' THEN txn_amount
         	ELSE 0
        END) as deposit_amt,
     SUM(CASE
         	WHEN txn_type = 'purchase' THEN txn_amount
         	ELSE 0
        END) as purchase_amt,
     SUM(CASE
         	WHEN txn_type = 'withdrawal' THEN txn_amount
         	ELSE 0
        END) as withdrawal_amt
    FROM data_bank.customer_transactions
    GROUP BY customer_id, month_num, month_name
    ORDER BY customer_id, month_num), 
    
    closing_balance_cte AS
    (SELECT customer_id, month_num, month_name, SUM(deposit_amt - purchase_amt - withdrawal_amt) OVER(PARTITION BY customer_id ORDER BY month_num) AS closing_balance
    FROM txn_cte
    ORDER BY customer_id, month_num),
    
    prev_balance_cte AS
    (SELECT *, LAG(closing_balance::INTEGER, 1, 0) OVER(PARTITION BY customer_id ORDER BY month_num) as prev_balance
    FROM closing_balance_cte)
    
    SELECT * 
    FROM prev_balance_cte
    WHERE prev_balance <> 0 AND
    closing_balance > 1.05 * prev_balance;

| customer_id | month_num | month_name | closing_balance | prev_balance |
| ----------- | --------- | ---------- | --------------- | ------------ |
| 2           | 3         | March      | 610             | 549          |
| 3           | 4         | April      | -729            | -1222        |
| 6           | 3         | March      | 340             | -52          |
| 7           | 2         | February   | 3173            | 964          |
| 9           | 3         | March      | 1584            | 654          |
| 10          | 2         | February   | -1342           | -1622        |
| 11          | 3         | March      | -2088           | -2469        |
| 12          | 3         | March      | 295             | 92           |
| 13          | 2         | February   | 1279            | 780          |
| 13          | 3         | March      | 1405            | 1279         |
| 14          | 2         | February   | 821             | 205          |
| 14          | 4         | April      | 989             | 821          |
| 15          | 4         | April      | 1102            | 379          |
| 16          | 4         | April      | -3422           | -4284        |
| 18          | 4         | April      | -815            | -842         |
| 19          | 4         | April      | 42              | -301         |
| 20          | 2         | February   | 519             | 465          |
| 20          | 3         | March      | 776             | 519          |
| 22          | 3         | March      | -149            | -1039        |
| 23          | 3         | March      | -156            | -314         |
| 24          | 2         | February   | 813             | 615          |
| 25          | 4         | April      | -304            | -1220        |
| 27          | 2         | February   | -713            | -1189        |
| 28          | 4         | April      | 272             | -1228        |
| 29          | 2         | February   | -76             | -138         |
| 29          | 3         | March      | 831             | -76          |
| 30          | 4         | April      | 508             | -431         |
| 32          | 2         | February   | 376             | -89          |
| 33          | 3         | March      | 1225            | -116         |
| 34          | 3         | March      | -185            | -347         |
| 36          | 2         | February   | 290             | 149          |
| 36          | 3         | March      | 1041            | 290          |
| 37          | 2         | February   | 902             | 85           |
| 37          | 4         | April      | -959            | -1069        |
| 39          | 2         | February   | 2388            | 1429         |
| 40          | 3         | March      | 659             | 295          |
| 41          | 2         | February   | 1379            | -46          |
| 41          | 3         | March      | 3441            | 1379         |
| 42          | 2         | February   | 1067            | 447          |
| 43          | 3         | March      | 869             | -406         |
| 44          | 2         | February   | -19             | -690         |
| 45          | 3         | March      | 584             | -1152        |
| 46          | 2         | February   | 1388            | 522          |
| 46          | 4         | April      | 104             | 80           |
| 50          | 3         | March      | 275             | -674         |
| 50          | 4         | April      | 450             | 275          |
| 51          | 3         | March      | 779             | -97          |
| 51          | 4         | April      | 1364            | 779          |
| 52          | 2         | February   | 2612            | 1140         |
| 53          | 2         | February   | 210             | 22           |
| 53          | 4         | April      | 227             | -728         |
| 54          | 4         | April      | 968             | 533          |
| 55          | 3         | March      | 349             | -410         |
| 58          | 2         | February   | 1697            | 383          |
| 58          | 4         | April      | -635            | -1196        |
| 59          | 2         | February   | 2190            | 924          |
| 60          | 2         | February   | 668             | -189         |
| 61          | 2         | February   | 323             | 222          |
| 67          | 2         | February   | 2565            | 1593         |
| 70          | 2         | February   | -63             | -584         |
| 74          | 3         | March      | 318             | 229          |
| 75          | 2         | February   | 294             | 234          |
| 76          | 2         | February   | 2081            | 925          |
| 77          | 2         | February   | 501             | 120          |
| 77          | 3         | March      | 797             | 501          |
| 78          | 3         | March      | -717            | -762         |
| 79          | 2         | February   | 1380            | 521          |
| 80          | 2         | February   | 1190            | 795          |
| 82          | 2         | February   | -3986           | -3912        |
| 82          | 3         | March      | -3249           | -3986        |
| 83          | 4         | April      | -377            | -742         |
| 85          | 3         | March      | 1076            | 467          |
| 86          | 3         | March      | 93              | -504         |
| 87          | 4         | April      | -1195           | -1563        |
| 88          | 2         | February   | 752             | -35          |
| 91          | 4         | April      | -2495           | -2660        |
| 93          | 2         | February   | 1103            | 399          |
| 93          | 3         | March      | 1186            | 1103         |
| 94          | 3         | March      | -1542           | -1496        |
| 95          | 2         | February   | 960             | 217          |
| 95          | 3         | March      | 1446            | 960          |
| 96          | 2         | February   | 1537            | 1048         |
| 98          | 4         | April      | 750             | -95          |
| 102         | 2         | February   | 1428            | 917          |
| 102         | 3         | March      | 1865            | 1428         |
| 104         | 2         | February   | 1087            | 615          |
| 104         | 3         | March      | 1190            | 1087         |
| 106         | 2         | February   | 846             | -109         |
| 108         | 2         | February   | 738             | 530          |
| 108         | 3         | March      | 1546            | 738          |
| 108         | 4         | April      | 2680            | 1546         |
| 109         | 2         | February   | 2491            | 429          |
| 110         | 3         | March      | 2233            | 1198         |
| 111         | 2         | February   | 463             | 101          |
| 113         | 2         | February   | 62              | -511         |
| 114         | 4         | April      | 1143            | 169          |
| 115         | 3         | March      | 884             | -845         |
| 116         | 3         | March      | 543             | 53           |
| 118         | 2         | February   | -513            | -683         |
| 119         | 4         | April      | -490            | -907         |
| 120         | 2         | February   | 1913            | 824          |
| 122         | 3         | March      | 1347            | 252          |
| 123         | 3         | March      | -1584           | -2277        |
| 124         | 2         | February   | 1878            | 731          |
| 125         | 3         | March      | -2436           | -2479        |
| 126         | 2         | February   | -716            | -786         |
| 127         | 2         | February   | 703             | 217          |
| 127         | 4         | April      | 1672            | 703          |
| 128         | 4         | April      | -202            | -776         |
| 129         | 3         | March      | 68              | -796         |
| 130         | 3         | March      | 132             | -1160        |
| 131         | 3         | March      | -152            | -983         |
| 133         | 2         | February   | -368            | -356         |
| 135         | 2         | February   | 977             | 104          |
| 136         | 2         | February   | 966             | 479          |
| 139         | 2         | February   | 504             | 44           |
| 139         | 3         | March      | 537             | 504          |
| 140         | 2         | February   | 1526            | 803          |
| 140         | 3         | March      | 2345            | 1526         |
| 141         | 2         | February   | 1483            | -369         |
| 141         | 3         | March      | 2113            | 1483         |
| 141         | 4         | April      | 2538            | 2113         |
| 142         | 4         | April      | 863             | 217          |
| 143         | 2         | February   | 1625            | 807          |
| 144         | 3         | March      | -3046           | -3280        |
| 145         | 2         | February   | -1970           | -3051        |
| 146         | 4         | April      | -3717           | -3781        |
| 147         | 2         | February   | 1698            | 600          |
| 148         | 3         | March      | -2076           | -2467        |
| 153         | 2         | February   | -776            | -1954        |
| 154         | 3         | March      | -2104           | -2340        |
| 156         | 4         | April      | 312             | 82           |
| 157         | 3         | March      | 2766            | -611         |
| 161         | 2         | February   | -961            | -1121        |
| 161         | 3         | March      | -291            | -961         |
| 162         | 2         | February   | 784             | 123          |
| 163         | 4         | April      | -3055           | -3116        |
| 164         | 2         | February   | 957             | 548          |
| 166         | 2         | February   | 1546            | 957          |
| 166         | 4         | April      | 1783            | 1303         |
| 167         | 2         | February   | 574             | 51           |
| 169         | 3         | March      | 9               | -1190        |
| 169         | 4         | April      | 906             | 9            |
| 170         | 3         | March      | -137            | -373         |
| 171         | 4         | April      | -911            | -1921        |
| 173         | 2         | February   | 1398            | 1298         |
| 174         | 4         | April      | 644             | -1135        |
| 175         | 4         | April      | -1549           | -1822        |
| 177         | 3         | March      | 800             | -156         |
| 178         | 2         | February   | 387             | 252          |
| 182         | 4         | April      | -159            | -843         |
| 185         | 4         | April      | -505            | -1001        |
| 186         | 2         | February   | 1345            | 534          |
| 186         | 3         | March      | 1930            | 1345         |
| 187         | 4         | April      | -2272           | -3060        |
| 188         | 2         | February   | 1013            | -184         |
| 190         | 2         | February   | 459             | 14           |
| 190         | 3         | March      | 523             | 459          |
| 190         | 4         | April      | 1178            | 523          |
| 192         | 3         | March      | 383             | -689         |
| 192         | 4         | April      | 1139            | 383          |
| 194         | 3         | March      | 178             | -2211        |
| 196         | 2         | February   | 1295            | 734          |
| 196         | 3         | March      | 1382            | 1295         |
| 197         | 2         | February   | 137             | -446         |
| 197         | 3         | March      | 1023            | 137          |
| 197         | 4         | April      | 3685            | 1023         |
| 198         | 4         | April      | -757            | -1253        |
| 200         | 2         | February   | 1356            | 997          |
| 200         | 3         | March      | 2914            | 1356         |
| 201         | 2         | February   | -292            | -383         |
| 201         | 3         | March      | 1529            | -292         |
| 203         | 2         | February   | 3471            | 2528         |
| 204         | 2         | February   | 1039            | 749          |
| 204         | 3         | March      | 1587            | 1039         |
| 204         | 4         | April      | 1893            | 1587         |
| 205         | 2         | February   | 1211            | -82          |
| 207         | 4         | April      | -1014           | -2152        |
| 208         | 4         | April      | 1361            | 406          |
| 210         | 3         | March      | -1309           | -1361        |
| 210         | 4         | April      | -792            | -1309        |
| 211         | 2         | February   | 1839            | 607          |
| 212         | 2         | February   | 481             | -336         |
| 212         | 3         | March      | 3529            | 481          |
| 213         | 3         | March      | -1184           | -1199        |
| 214         | 3         | March      | 83              | -1511        |
| 214         | 4         | April      | 802             | 83           |
| 215         | 2         | February   | 1770            | 822          |
| 216         | 2         | February   | 3302            | 1619         |
| 217         | 2         | February   | 1839            | 870          |
| 218         | 3         | March      | -465            | -1620        |
| 218         | 4         | April      | 1167            | -465         |
| 219         | 3         | March      | 1108            | -845         |
| 220         | 2         | February   | 714             | 307          |
| 221         | 2         | February   | 1481            | 1384         |
| 222         | 2         | February   | 1997            | 657          |
| 222         | 4         | April      | 1532            | 1136         |
| 224         | 4         | April      | -1369           | -1581        |
| 225         | 3         | March      | 297             | -89          |
| 226         | 2         | February   | -586            | -980         |
| 226         | 4         | April      | -1430           | -1855        |
| 229         | 2         | February   | 1618            | 621          |
| 230         | 2         | February   | 990             | 499          |
| 230         | 3         | March      | 2428            | 990          |
| 232         | 3         | March      | 923             | 864          |
| 233         | 2         | February   | 2910            | 1795         |
| 233         | 3         | March      | 3742            | 2910         |
| 234         | 2         | February   | 322             | -200         |
| 235         | 2         | February   | -476            | -1963        |
| 235         | 3         | March      | 24              | -476         |
| 236         | 2         | February   | 1059            | 356          |
| 237         | 2         | February   | 136             | -174         |
| 237         | 3         | March      | 1567            | 136          |
| 238         | 2         | February   | 1270            | 802          |
| 239         | 2         | February   | 706             | -10          |
| 239         | 4         | April      | 1871            | 574          |
| 240         | 3         | March      | 3200            | 754          |
| 243         | 3         | March      | 979             | -368         |
| 244         | 2         | February   | 1752            | 728          |
| 244         | 3         | March      | 1930            | 1752         |
| 245         | 4         | April      | -1021           | -1791        |
| 246         | 2         | February   | 584             | 506          |
| 248         | 2         | February   | 667             | 304          |
| 248         | 3         | March      | 955             | 667          |
| 248         | 4         | April      | 1188            | 955          |
| 249         | 3         | March      | 1065            | 336          |
| 253         | 2         | February   | -577            | -578         |
| 253         | 3         | March      | -457            | -577         |
| 254         | 3         | March      | -590            | -2883        |
| 256         | 3         | March      | 1152            | 906          |
| 257         | 3         | March      | -472            | -1609        |
| 258         | 4         | April      | -1465           | -2893        |
| 261         | 2         | February   | 1408            | 746          |
| 261         | 4         | April      | -31             | -329         |
| 262         | 3         | March      | -2372           | -2599        |
| 263         | 4         | April      | 770             | 112          |
| 264         | 2         | February   | 1545            | 770          |
| 264         | 3         | March      | 2088            | 1545         |
| 265         | 4         | April      | -1948           | -2592        |
| 266         | 2         | February   | 1455            | 651          |
| 266         | 4         | April      | 1138            | 787          |
| 267         | 4         | April      | -2442           | -5236        |
| 268         | 3         | March      | 270             | 123          |
| 269         | 3         | March      | -2470           | -3985        |
| 269         | 4         | April      | -1864           | -2470        |
| 271         | 2         | February   | 272             | -1586        |
| 271         | 4         | April      | 180             | -616         |
| 273         | 4         | April      | 308             | -178         |
| 274         | 2         | February   | -582            | -780         |
| 274         | 3         | March      | 124             | -582         |
| 276         | 2         | February   | -796            | -851         |
| 277         | 2         | February   | 1411            | 615          |
| 277         | 3         | March      | 1865            | 1411         |
| 278         | 2         | February   | 1723            | 1309         |
| 278         | 3         | March      | 2026            | 1723         |
| 278         | 4         | April      | 3554            | 2026         |
| 279         | 2         | February   | 3641            | 1895         |
| 279         | 3         | March      | 4183            | 3641         |
| 281         | 2         | February   | 1055            | 220          |
| 281         | 3         | March      | 2192            | 1055         |
| 281         | 4         | April      | 3004            | 2192         |
| 282         | 4         | April      | -1311           | -1661        |
| 285         | 2         | February   | 1358            | 360          |
| 285         | 3         | March      | 1965            | 1358         |
| 287         | 2         | February   | 829             | 658          |
| 287         | 3         | March      | 886             | 829          |
| 288         | 3         | March      | -515            | -867         |
| 290         | 2         | February   | 1139            | 785          |
| 290         | 3         | March      | 2061            | 1139         |
| 292         | 3         | March      | -4760           | -4646        |
| 294         | 2         | February   | 1557            | 307          |
| 295         | 3         | March      | 1430            | 496          |
| 296         | 2         | February   | 1152            | 191          |
| 296         | 3         | March      | 1309            | 1152         |
| 296         | 4         | April      | 2220            | 1309         |
| 297         | 2         | February   | 585             | 550          |
| 297         | 3         | March      | 1004            | 585          |
| 297         | 4         | April      | 1282            | 1004         |
| 298         | 3         | March      | 767             | -580         |
| 298         | 4         | April      | 1489            | 767          |
| 299         | 2         | February   | 1246            | 961          |
| 301         | 4         | April      | -3529           | -3636        |
| 302         | 4         | April      | -1795           | -2410        |
| 303         | 2         | February   | 465             | 332          |
| 303         | 4         | April      | -660            | -629         |
| 304         | 4         | April      | -2047           | -2420        |
| 305         | 2         | February   | 189             | 20           |
| 307         | 2         | February   | 750             | -696         |
| 308         | 2         | February   | 316             | -561         |
| 308         | 3         | March      | 710             | 316          |
| 308         | 4         | April      | 971             | 710          |
| 309         | 3         | March      | -1813           | -2404        |
| 309         | 4         | April      | -960            | -1813        |
| 310         | 3         | March      | 3066            | 156          |
| 311         | 2         | February   | 1006            | 310          |
| 312         | 2         | February   | 656             | 485          |
| 313         | 2         | February   | 972             | 901          |
| 313         | 4         | April      | -34             | -1311        |
| 314         | 3         | March      | 91              | -633         |
| 315         | 3         | March      | 2287            | 1349         |
| 317         | 2         | February   | 1232            | 869          |
| 319         | 3         | March      | -488            | -703         |
| 320         | 3         | March      | 2239            | 1909         |
| 320         | 4         | April      | 2851            | 2239         |
| 321         | 4         | April      | 572             | -213         |
| 322         | 2         | February   | 2471            | 1949         |
| 324         | 2         | February   | 967             | 203          |
| 324         | 3         | March      | 1470            | 967          |
| 325         | 3         | March      | -1862           | -1878        |
| 326         | 2         | February   | 417             | -211         |
| 328         | 4         | April      | -4559           | -4703        |
| 329         | 4         | April      | 736             | -626         |
| 330         | 2         | February   | 1099            | 826          |
| 332         | 2         | February   | 559             | 202          |
| 332         | 4         | April      | 670             | 494          |
| 333         | 2         | February   | -127            | -229         |
| 333         | 3         | March      | 567             | -127         |
| 333         | 4         | April      | 920             | 567          |
| 334         | 2         | February   | 2724            | 1177         |
| 335         | 3         | March      | 423             | -354         |
| 336         | 3         | March      | 135             | -595         |
| 336         | 4         | April      | 599             | 135          |
| 337         | 2         | February   | 170             | -264         |
| 337         | 3         | March      | 1450            | 170          |
| 338         | 2         | February   | 533             | 262          |
| 338         | 3         | March      | 2767            | 533          |
| 339         | 2         | February   | 1088            | -780         |
| 340         | 2         | February   | 276             | -1086        |
| 340         | 3         | March      | 559             | 276          |
| 340         | 4         | April      | 1390            | 559          |
| 341         | 4         | April      | -2094           | -2133        |
| 342         | 3         | March      | 503             | -285         |
| 343         | 2         | February   | 1653            | 1339         |
| 343         | 3         | March      | 1841            | 1653         |
| 344         | 2         | February   | 505             | -932         |
| 344         | 3         | March      | 1475            | 505          |
| 348         | 2         | February   | 114             | -771         |
| 348         | 4         | April      | 48              | -155         |
| 349         | 3         | March      | 1309            | -1040        |
| 349         | 4         | April      | 1964            | 1309         |
| 353         | 3         | March      | -516            | -1819        |
| 355         | 2         | February   | -194            | -245         |
| 355         | 4         | April      | -852            | -1137        |
| 357         | 2         | February   | 878             | 780          |
| 358         | 2         | February   | -619            | -1062        |
| 358         | 4         | April      | -708            | -883         |
| 359         | 2         | February   | 1284            | 890          |
| 359         | 3         | March      | 3159            | 1284         |
| 360         | 2         | February   | -427            | -1306        |
| 360         | 4         | April      | -324            | -1255        |
| 361         | 2         | February   | 772             | 340          |
| 362         | 2         | February   | 481             | 416          |
| 362         | 4         | April      | 615             | -564         |
| 363         | 4         | April      | -2886           | -3065        |
| 364         | 3         | March      | 1173            | -456         |
| 365         | 2         | February   | 244             | -68          |
| 366         | 4         | April      | -1096           | -1305        |
| 367         | 4         | April      | -1697           | -3101        |
| 368         | 3         | March      | -1476           | -3490        |
| 369         | 3         | March      | 1679            | 266          |
| 370         | 2         | February   | -1635           | -2295        |
| 370         | 3         | March      | -1311           | -1635        |
| 371         | 3         | March      | -5              | -154         |
| 373         | 3         | March      | 1057            | 277          |
| 373         | 4         | April      | 1451            | 1057         |
| 374         | 3         | March      | -846            | -1292        |
| 376         | 2         | February   | 2515            | 1614         |
| 376         | 3         | March      | 3062            | 2515         |
| 378         | 2         | February   | 2424            | 484          |
| 381         | 2         | February   | 992             | 66           |
| 382         | 3         | March      | -1141           | -1195        |
| 383         | 2         | February   | 935             | -36          |
| 383         | 4         | April      | 913             | -617         |
| 384         | 3         | March      | -2527           | -2486        |
| 385         | 4         | April      | -4861           | -4693        |
| 387         | 3         | March      | 2551            | 1069         |
| 388         | 3         | March      | 1598            | 1126         |
| 389         | 2         | February   | 490             | -27          |
| 389         | 3         | March      | 1214            | 490          |
| 389         | 4         | April      | 2005            | 1214         |
| 390         | 3         | March      | -1929           | -2038        |
| 391         | 3         | March      | 272             | 2            |
| 392         | 2         | February   | 2034            | 816          |
| 393         | 3         | March      | 1500            | 118          |
| 395         | 2         | February   | -914            | -1782        |
| 395         | 3         | March      | -903            | -914         |
| 396         | 2         | February   | -550            | -909         |
| 397         | 2         | February   | 1106            | 973          |
| 397         | 3         | March      | 1709            | 1106         |
| 400         | 4         | April      | 1338            | -409         |
| 401         | 3         | March      | 83              | -25          |
| 402         | 2         | February   | 1599            | 1478         |
| 403         | 3         | March      | 987             | 303          |
| 403         | 4         | April      | 1047            | 987          |
| 405         | 4         | April      | -7070           | -6882        |
| 406         | 2         | February   | 1131            | 795          |
| 406         | 3         | March      | 2597            | 1131         |
| 407         | 2         | February   | 46              | 7            |
| 408         | 3         | March      | 800             | -145         |
| 409         | 2         | February   | 1371            | 155          |
| 409         | 3         | March      | 2455            | 1371         |
| 409         | 4         | April      | 2623            | 2455         |
| 413         | 2         | February   | 1072            | 642          |
| 414         | 3         | March      | 1918            | 439          |
| 416         | 2         | February   | 1715            | 756          |
| 416         | 3         | March      | 3324            | 1715         |
| 416         | 4         | April      | 3898            | 3324         |
| 420         | 4         | April      | -2078           | -2457        |
| 421         | 2         | February   | -571            | -741         |
| 421         | 3         | March      | -413            | -571         |
| 421         | 4         | April      | 312             | -413         |
| 422         | 3         | March      | -1046           | -1661        |
| 423         | 3         | March      | 1321            | -262         |
| 424         | 4         | April      | -314            | -940         |
| 425         | 3         | March      | 57              | -505         |
| 427         | 2         | February   | 1305            | 588          |
| 428         | 2         | February   | 687             | 280          |
| 428         | 3         | March      | 1217            | 687          |
| 429         | 2         | February   | 473             | 82           |
| 430         | 2         | February   | 403             | -8           |
| 432         | 2         | February   | 986             | 392          |
| 432         | 4         | April      | 2435            | 963          |
| 433         | 2         | February   | 1286            | 883          |
| 434         | 4         | April      | -1815           | -2366        |
| 435         | 2         | February   | -38             | -1329        |
| 437         | 3         | March      | -1318           | -1537        |
| 437         | 4         | April      | -1134           | -1318        |
| 438         | 2         | February   | 2813            | 1317         |
| 439         | 4         | April      | 318             | -381         |
| 440         | 2         | February   | 146             | -123         |
| 440         | 3         | March      | 490             | 146          |
| 441         | 2         | February   | 745             | -329         |
| 443         | 4         | April      | -11             | -359         |
| 444         | 3         | March      | -499            | -1178        |
| 446         | 4         | April      | 405             | -53          |
| 449         | 3         | March      | -3045           | -3928        |
| 450         | 4         | April      | -36             | -737         |
| 451         | 3         | March      | -645            | -1313        |
| 452         | 2         | February   | 1654            | 1360         |
| 453         | 2         | February   | 811             | 638          |
| 453         | 4         | April      | 117             | -595         |
| 454         | 2         | February   | 2163            | 11           |
| 456         | 4         | April      | 12              | -55          |
| 457         | 4         | April      | -718            | -714         |
| 459         | 3         | March      | -2834           | -2912        |
| 460         | 3         | March      | -1175           | -1158        |
| 460         | 4         | April      | -327            | -1175        |
| 461         | 2         | February   | 3431            | 2267         |
| 463         | 3         | March      | 673             | 312          |
| 465         | 2         | February   | 1989            | 955          |
| 467         | 2         | February   | 3582            | 1994         |
| 469         | 2         | February   | 2161            | 386          |
| 471         | 3         | March      | 1238            | 781          |
| 471         | 4         | April      | 1887            | 1238         |
| 472         | 3         | March      | 32              | -115         |
| 472         | 4         | April      | 218             | 32           |
| 478         | 2         | February   | 2278            | -712         |
| 479         | 3         | March      | 513             | -327         |
| 480         | 4         | April      | -165            | -235         |
| 483         | 4         | April      | 1330            | -189         |
| 484         | 3         | March      | 1796            | 871          |
| 485         | 2         | February   | 1507            | 16           |
| 485         | 3         | March      | 2202            | 1507         |
| 487         | 2         | February   | 312             | -572         |
| 488         | 2         | February   | 297             | -243         |
| 488         | 4         | April      | -191            | -412         |
| 489         | 2         | February   | 1808            | 556          |
| 489         | 3         | March      | 3342            | 1808         |
| 489         | 4         | April      | 5338            | 3342         |
| 490         | 2         | February   | 342             | 271          |
| 491         | 2         | February   | 298             | -3           |
| 493         | 4         | April      | -738            | -824         |
| 494         | 2         | February   | 909             | 529          |
| 494         | 3         | March      | 1447            | 909          |
| 495         | 3         | March      | -89             | -1438        |
| 496         | 3         | March      | -2426           | -3076        |
| 497         | 2         | February   | 1003            | 754          |
| 497         | 3         | March      | 1739            | 1003         |
| 497         | 4         | April      | 2680            | 1739         |
| 498         | 2         | February   | 2195            | 1360         |
| 498         | 3         | March      | 2989            | 2195         |
| 498         | 4         | April      | 3488            | 2989         |
| 499         | 2         | February   | 1415            | -304         |
| 500         | 2         | February   | 2981            | 1594         |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/2GtQz4wZtuNNu7zXH5HtV4/3)
