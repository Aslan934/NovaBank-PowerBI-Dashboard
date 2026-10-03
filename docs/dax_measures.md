# DAX Ölçüləri — NovaBank Power BI Dashboard

Bu sənəd `Metrics` cədvəlindəki bütün DAX ölçülərini və onların məntiqini izah edir. Maliyyə ölçülərinin əksəriyyəti yalnız **Approved** statuslu əməliyyatlar üzərində hesablanır (əks halda qeyd olunub).

Əsas fakt cədvəli: `Transactions_2024_2025` (status sütunu: `Status`).

---

## 1. Core Financials

### Revenue_Approved
Təsdiqlənmiş əməliyyatların ümumi dövriyyəsi (həcmi).
```dax
Revenue_Approved = 
CALCULATE(
    SUM(Transactions_2024_2025[Amount]),
    Transactions_2024_2025[Status] = "Approved"
)
```

### Commission_Revenue
```dax
Commission_Revenue = 
CALCULATE(
    SUM(Transactions_2024_2025[FeeRevenue]),
    Transactions_2024_2025[Status] = "Approved"
)
```

### Interchange_Revenue
```dax
Interchange_Revenue = 
CALCULATE(
    SUM(Transactions_2024_2025[InterchangeRevenue]),
    Transactions_2024_2025[Status] = "Approved"
)
```

### Total_Revenue
Ümumi gəlir = komissiya + interchange.
```dax
Total_Revenue = [Commission_Revenue] + [Interchange_Revenue]
```

### Total_Cost
Ümumi xərc = cashback + processing + fraud.
```dax
Total_Cost = 
CALCULATE(
    SUM(Transactions_2024_2025[CashbackCost]) +
    SUM(Transactions_2024_2025[ProcessingCost]) +
    SUM(Transactions_2024_2025[FraudLoss]),
    Transactions_2024_2025[Status] = "Approved"
)
```

### Net_Operating_Income
Xalis əməliyyat gəliri.
```dax
Net_Operating_Income = [Total_Revenue] - [Total_Cost]
```

---

## 2. Ratios

### Net_Margin_%
Xalis gəlirin dövriyyəyə nisbəti (Total_Revenue-ya yox, **dövriyyəyə** bölünür).
```dax
Net_Margin_% = DIVIDE([Net_Operating_Income], [Revenue_Approved])
```

### Cost_to_Income
Xərcin gəlirə nisbəti — 100%-i keçərsə, o dövr zərərlədir.
```dax
Cost_to_Income = DIVIDE([Total_Cost], [Total_Revenue])
```

### Weighted_Cashback_%
Çəkili cashback faizi (sadə orta yox, məbləğə görə çəkilənmiş nisbət).
```dax
Weighted_Cashback_% = 
DIVIDE(
    CALCULATE(SUM(Transactions_2024_2025[CashbackCost]), Transactions_2024_2025[Status] = "Approved"),
    [Revenue_Approved]
)
```

### Fraud_Loss_Rate
⚠️ **Xüsusi qayda:** `FraudLoss` dəyəri Approved statuslu sətirlərdə həmişə 0-dır (fraud aşkarlananda əməliyyat rədd olunur). Ona görə bu ölçü **statusdan asılı olmayaraq, bütün əməliyyatlar üzrə** hesablanır — digər nisbət ölçülərindən fərqli olaraq, heç bir `CALCULATE`/status filtri yoxdur.
```dax
Fraud_Loss_Rate = 
DIVIDE(
    SUM(Transactions_2024_2025[FraudLoss]),
    SUM(Transactions_2024_2025[Amount])
)
```

---

## 3. Activity

### Approved_Count
```dax
Approved_Count = 
CALCULATE(
    COUNTROWS(Transactions_2024_2025),
    Transactions_2024_2025[Status] = "Approved"
)
```

### Active_Customers
Ən azı 1 Approved əməliyyatı olan unikal müştəri sayı.
```dax
Active_Customers = 
CALCULATE(
    DISTINCTCOUNT(Transactions_2024_2025[CustomerID]),
    Transactions_2024_2025[Status] = "Approved"
)
```

### Avg_Transaction_Amount
```dax
Avg_Transaction_Amount = 
CALCULATE(
    AVERAGE(Transactions_2024_2025[Amount]),
    Transactions_2024_2025[Status] = "Approved"
)
```

### Decline_Rate
Məxrəc — cari filterdəki **bütün statuslar** üzrə əməliyyat sayı (status filtri qoyulmur).
```dax
Decline_Rate = 
DIVIDE(
    CALCULATE(COUNTROWS(Transactions_2024_2025), Transactions_2024_2025[Status] = "Declined"),
    COUNTROWS(Transactions_2024_2025)
)
```

### Repeat_Customer_Count
Cari dövrdə ən azı 5 Approved əməliyyatı olan müştəri sayı.
```dax
Repeat_Customer_Count = 
COUNTROWS(
    FILTER(
        SUMMARIZECOLUMNS(
            Transactions_2024_2025[CustomerID],
            "ApprovedCount", CALCULATE(
                COUNTROWS(Transactions_2024_2025),
                Transactions_2024_2025[Status] = "Approved"
            )
        ),
        [ApprovedCount] >= 5
    )
)
```

### Repeat_Customer_%
Məxrəc — cari dövrdəki **bütün unikal müştərilər** (statusdan asılı olmayaraq).
```dax
Repeat_Customer_% = 
DIVIDE(
    [Repeat_Customer_Count],
    DISTINCTCOUNT(Transactions_2024_2025[CustomerID])
)
```

---

## 4. Time Intelligence

> Bu bölmə `Calendar` cədvəlinin **Date Table** kimi işarələnməsini və `Calendar[Date]` ↔ `Transactions_2024_2025[TransactionDate]` əlaqəsinin aktiv olmasını tələb edir.

### Revenue_Approved_PY
Keçən ilin eyni dövrü.
```dax
Revenue_Approved_PY = 
CALCULATE(
    [Revenue_Approved],
    SAMEPERIODLASTYEAR(Calendar[Date])
)
```

### Net_Operating_Income_PY
```dax
Net_Operating_Income_PY = 
CALCULATE(
    [Net_Operating_Income],
    SAMEPERIODLASTYEAR(Calendar[Date])
)
```

### Revenue_Approved_YoY / _YoY_%
Əvvəlki dövr yoxdursa (PY = BLANK), nəticə BLANK qalır.
```dax
Revenue_Approved_YoY = 
IF(
    ISBLANK([Revenue_Approved_PY]),
    BLANK(),
    [Revenue_Approved] - [Revenue_Approved_PY]
)

Revenue_Approved_YoY_% = 
DIVIDE([Revenue_Approved_YoY], [Revenue_Approved_PY])
```

### Net_Operating_Income_YoY / _YoY_%
```dax
Net_Operating_Income_YoY = 
IF(
    ISBLANK([Net_Operating_Income_PY]),
    BLANK(),
    [Net_Operating_Income] - [Net_Operating_Income_PY]
)

Net_Operating_Income_YoY_% = 
DIVIDE([Net_Operating_Income_YoY], [Net_Operating_Income_PY])
```

### Revenue_Approved_YTD / Net_Operating_Income_YTD
İlin əvvəlindən yığılan (cumulative) dəyər.
```dax
Revenue_Approved_YTD = TOTALYTD([Revenue_Approved], Calendar[Date])

Net_Operating_Income_YTD = TOTALYTD([Net_Operating_Income], Calendar[Date])
```

### Net_Income_MA3
Son 3 ayın hərəkətli orta xalis gəliri (vizualda ay səviyyəsində istifadə olunmalıdır).
```dax
Net_Income_MA3 = 
AVERAGEX(
    DATESINPERIOD(
        Calendar[Date],
        MAX(Calendar[Date]),
        -3,
        MONTH
    ),
    [Net_Operating_Income]
)
```

---

## 5. Target (Plan vs Faktiki)

> `Targets[MonthStart]` ↔ `Calendar[MonthStart]` əlaqəsi tələb olunur (hər ikisi Date tipində olmalıdır).

### Net_Income_Target
```dax
Net_Income_Target = SUM(Targets[NetIncomeTarget])
```

### Target_Variance / Target_Variance_%
```dax
Target_Variance = [Net_Operating_Income] - [Net_Income_Target]

Target_Variance_% = DIVIDE([Target_Variance], [Net_Income_Target])
```

### Target_Achievement_%
100% = hədəfə tam çatılıb.
```dax
Target_Achievement_% = DIVIDE([Net_Operating_Income], [Net_Income_Target])
```

---

## 6. Scenario (What-If) — Əlavə Cashback

### Extra Cashback % (parametr)
`Modeling → New Parameter → Numeric range`, 0–0.03 (0%–3%), addım 0.0025.
```dax
Extra Cashback % = GENERATESERIES(0, 0.03, 0.0025)
```
Avtomatik yaranan ölçü: `Extra Cashback % Value` — slайserdə seçilən dəyəri qaytarır.

### Extra_Cashback_Cost
Əlavə cashback yalnız Approved dövriyyəyə tətbiq olunur, mövcud `CashbackCost`-un üzərinə **əlavə** gəlir (onu əvəz etmir).
```dax
Extra_Cashback_Cost = 
[Revenue_Approved] * 'Extra Cashback %'[Extra Cashback % Value]
```

### Total_Cost_Scenario
```dax
Total_Cost_Scenario = [Total_Cost] + [Extra_Cashback_Cost]
```

### Net_Operating_Income_Scenario
```dax
Net_Operating_Income_Scenario = [Total_Revenue] - [Total_Cost_Scenario]
```

### Breakeven_Cashback_%
Xalis gəlirin sıfıra düşdüyü əlavə cashback həddi.
```dax
Breakeven_Cashback_% = DIVIDE([Net_Operating_Income], [Revenue_Approved])
```

---

## 7. Advanced

### Top20_Customer_Share
Dövriyyəyə görə ilk 20% müştərinin ümumi dövriyyə payı (Pareto analizi).
```dax
Top20_Customer_Share = 
VAR CustomerRevenue = 
    SUMMARIZECOLUMNS(
        Transactions_2024_2025[CustomerID],
        "CustRevenue", [Revenue_Approved]
    )
VAR TotalCustomers = COUNTROWS(CustomerRevenue)
VAR Top20Count = ROUNDUP(TotalCustomers * 0.2, 0)
VAR RankedCustomers = 
    TOPN(
        Top20Count,
        CustomerRevenue,
        [CustRevenue],
        DESC
    )
VAR Top20Revenue = SUMX(RankedCustomers, [CustRevenue])
VAR TotalRevenue = [Revenue_Approved]
RETURN
    DIVIDE(Top20Revenue, TotalRevenue)
```

### Product_Share_of_Category
Məhsulun öz kateqoriyası daxilindəki dövriyyə payı. `REMOVEFILTERS` yalnız `ProductName` filtrini silir — `Category` və digər filtrlər (tarix, seqment, şəhər) qorunur. Bu ölçü Top N filtrindən asılı olmayaraq bütün məhsulları göstərməlidir.
```dax
Product_Share_of_Category = 
DIVIDE(
    [Revenue_Approved],
    CALCULATE(
        [Revenue_Approved],
        REMOVEFILTERS(Products[ProductName])
    )
)
```

---

## Qeydlər

- Bütün nisbət (`%`) ölçüləri Power BI-da **Percentage** formatına keçirilib.
- Mənfi `Net_Operating_Income` və mənfi marja dəyərləri dashboard-da **qırmızı** rənglə fərqləndirilir (conditional formatting, Font color → Rules).
- `Fraud_Loss_Rate` istisna olmaqla, bütün maliyyə ölçüləri `Status = "Approved"` filtri ilə hesablanır.
- Dataset NovaBank üçün hazırlanmış **sintetik tədris məlumatıdır**.
