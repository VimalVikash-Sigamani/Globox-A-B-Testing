# Globox A/B Test: Food and Drink Banner

### Project Overview
GloBox (fictional company) wants to bring awareness about food & drink product catgeory to increase the revenue, hence decided to add food & drink banner at top of the website.
Growth team decided to run A/B test in order to prove the website without banner(control group) or with banner (treatment group) which one is performs better to achieve the desired goal (increase in revenue).

### Data Source
Data used for analysis are stored in PostgreSQL database. Below are the tables details 
- **users:** user demographic information
  - **id:** the user ID
  - **country:** ISO 3166 alpha-3 country code
  - **gender:** the user's gender (M = male, F = female, O = other)
    
- **groups:** user A/B test group assignment
  - **uid:** the user ID
  - **group:** the user’s test group
  - **join_dt:** the date the user joined the test (visited the page)
  - **device:** the device the user visited the page on (I = iOS, A = android)

- **activity:** user purchase activity, containing 1 row per day that a user made a purchase
  - **uid:** the user ID
  - **dt:** date of purchase activity
  - **device:** the device type the user purchased on (I = iOS, A = android)
  - **spent:** the purchase amount in USD

### Tools
- PostgreSQL - Data Cleaning and Data Analysis
- Google spreadsheet - Statistical Analysis
- Tableau - Data Visualization

### Data Cleaning/Preparation
In the initial data preparation phase, we performed the following tasks
1. Data inspection and exploration.
2. Handling missing values.
3. Data cleaning & formatting.

### Exploratory Data Analysis (EDA)
EDA involved in exploring the A/B test data to answer some questions, such as:
1. What was the conversion rate of all users?
2. What is the user conversion rate for the control and treatment groups?
3. What is the average amount spent per user for the control and treatment groups, including users who did not convert?
4. Extract the user ID, user’s country, user’s gender, user’s device type, user’s test group, whether or not they converted (spent > $0), and how much they spent in total ($0+).

### Data Analysis

1. What was the conversion rate of all users?
   
   ```sql
   SELECT CONCAT(ROUND((purchase_count/CAST(total_user AS NUMERIC))*100,2),'%') AS total_conversion_rate
   FROM (
      SELECT COUNT(DISTINCT uid) AS purchase_count, CAST(COUNT(DISTINCT id) AS FLOAT) AS total_user
      FROM users
      LEFT JOIN activity
          ON users.id = activity.uid ) tbl;
   ```
2. What is the user conversion rate for the control and treatment groups?

   ```sql
   WITH CTE1 AS (
   SELECT grp, (CAST(purchase_count AS FLOAT)/CAST(total_users AS FLOAT))*100 AS conv_rate
   FROM (
         SELECT groups.group as grp, COUNT(DISTINCT id) AS total_users, COUNT(DISTINCT activity.uid) AS purchase_count
         FROM users
         JOIN groups
             ON users.id = groups.uid
         LEFT JOIN activity
             ON users.id = activity.uid
         GROUP BY groups.group ) tbl
   )

   SELECT grp, CONCAT(ROUND(CAST(conv_rate AS NUMERIC), 2), '%') AS conversion_rate
   FROM CTE1;
   ```
3. What is the average amount spent per user for the control and treatment groups, including users who did not convert?

   ```sql
   SELECT grp, ROUND(AVG(total_amt),2) AS average_amount_spent
   FROM (
         SELECT groups.uid, groups.group as grp, SUM(COALESCE(spent,0)) total_amt
         FROM groups
         LEFT JOIN activity
             ON groups.uid = activity.uid
         GROUP BY groups.uid, groups.group ) tbl
   GROUP BY grp
   ORDER BY grp;
   ```
4. Extract the user ID, user’s country, user’s gender, user’s device type, user’s test group, whether or not they converted (spent > $0), and how much they spent in total ($0+).

   ```sql
   WITH CTE1 AS (
   SELECT id, ROUND(SUM(spent_amt),2) AS Total_amount_spent
   FROM (
         SELECT id, COALESCE(spent, 0) AS spent_amt
         FROM users
         LEFT JOIN activity
             ON users.id = activity.uid ) tbl
   GROUP BY id
   ),

   CTE2 AS (
        SELECT *
        ,CASE WHEN total_amount_spent = 0 THEN 0 ELSE 1 END AS Converted
        FROM CTE1
   )

   SELECT CTE2.id, 
	        COALESCE(country, 'OTHER') AS cleaned_country, 
		      COALESCE(gender, 'MISSING') AS cleaned_gender, 
  	      COALESCE(device, 'MISSING') AS cleaned_device, 
          grp.group,
          converted, 
          total_amount_spent as total_spent
   FROM CTE2 
   JOIN users
	   	 ON CTE2.id = users.id
   JOIN groups grp
		   ON CTE2.id = grp.uid;
   ```

### Statistical Analysis
We measured below two key metrices and Hypothesis testing is done on these metrices to determine the outcome of A/B testing.

1. Conversion rate:
   -  Null Hypothesis H0: P1 = P2
   -  Alternative Hypothesis H1: P1 != P2

2. Average amount spent:
	 - Null Hypothesis H0: u1 = u2
   - Alternative Hypothesis H1: u1 != u2

Google sheet is used to perform Hypothesis testing and Confidence interval calculation using Z-TEST and T-TEST model. [Click here](https://docs.google.com/spreadsheets/d/1GU7lqwdvYfTGoxPXMLb5SucVjKEtxCC-vlK9KNT1xH8/edit#gid=392375140) for detailed calculations.

 **Conversion rate:**
- Z-Test is used to measure the Conversion rate.
- Pooled Sample Proportion = 0.04278446356 
- Standard Error = 0.001829526081 
- Test Statistics = -3.86429177
- P-Value = 0.0001114119853

For conversion rate metric, P=0.0001 is less than significance thershold limit (0.05) and statistically significant. Hence we reject the null hypothesis and proved that there is a difference in user conversion rate between the control and treatment groups.

With 95% Confidence level we determined that difference in conversion rate will falls between 0.0035% and 0.0107%.

**Average amount spent:**
- T-Test is used to measure the Average amount spent.
- Mean of group A = 3.37451752 
- Mean of group B = 3.390866667
- Standard Deviation of group A = 25.93638327 
- Standard Deviation of group B = 25.41410528 
- Standard Error = 0.2321405061 
- Test Statistics = -0.07042780472
- Degree of Freedom = 24342 
- P-Value = 0.9438537398

For average amount spent metric, P=0.944 is not less than significance thershold limit (0.05) and statistically insignificant. Hence we fail to reject the null hypothesis and proved that there is no difference in average amount spent per user between the control and treatment groups.

With 95% Confidence level we determined that difference in average spending of users will falls between -0.4387% and 0.4715%

### Results/Findings
Statistical evidence shows that there is significant increase in conversion rate, hence its proves that users who saw banner at top of the website are tend to make more purchase than the users who didn't see the banner. So adding the banner will increase the purchase.

On other hand, statistical evidence shows that there is no significant increase in average amount spend by the users, its proves that the banner doesn't influence the users to spend more on their purchase. 

Eventhough number of purchase are increased, amount spend on the purchase is not increased. Hence there is no increase in reveneue.

### Recommedations
Business to iterate the experiment for longer duration and with large number of users which will provide us more reliable results.

