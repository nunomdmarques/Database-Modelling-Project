# Database Modelling Project

## 1. Project Overview

For this project, I was tasked with developing a methodology to estimate **Monthly Active Users (MAUs)** for individual video game titles across various global markets. This includes defining access rates, identifying what data points to extract from multiple API endpoints, and combining this data with internal sources to create accurate and scalable MAU estimates.

The challenge was to combine user activity data, title metadata, and platform install base statistics while accounting for privacy limitations and sampling biases. The overarching goal was to produce daily MAU estimates to support a wide range of reporting and analytical needs, including trend analysis and market comparisons.

My role involved:
- Planning the data extraction,
- Database design, and 
- Estimation methodology.

## 2. Challenges and Requirements

### Key Challenges:
- **API Limitations & Privacy Restrictions**: Data was available for only 1 in 4 users, requiring a careful sampling strategy.
- **Time Zone Differences**: Multiple daily data requests were needed to align with different markets' peak activity times.
- **Large-Scale Data Handling**: The database had to handle thousands of titles across numerous markets while maintaining scalability and accuracy.
- **Sampling Biases**: Robust stratified sampling was required to ensure representativity across markets and genres.
- **Data Discrepancies**: Handling differences between system estimates and factual market data required constant model adjustments.

### Project Requirements:
- Real-time updates for MAU estimates.
- Effective sampling strategies to ensure data accuracy.
- Efficient database design and query performance optimization.

## 3. Data Sources

The project utilized several API endpoints and internal data sources:

- **User Activity Endpoint**: Provides title_id for active gaming sessions, accounting for privacy restrictions.
- **User Metadata Endpoint**: Contains user-specific data like user_id and country_code for geographic representation.
- **Title Metadata Endpoint**: Information on video game titles like title_id, name, and genre.
- **Install Base Data**: Internal data on the active user base, crucial for scaling MAUs.

## 4. Approach and Solution

### 4.1 Data Extraction
The following data extraction process was implemented:
- **User Activity Endpoint**: Data retrieved every 30 minutes to capture gaming sessions across time zones.
- **User Metadata & Title Metadata Endpoints**: Data retrieved daily to account for new users and games.

### 4.2 Database Design
A relational database was designed to handle large volumes of data efficiently.

| Table Name     | Primary Key                      | Foreign Key                      | Constraints                          | Relationships                                                     |
| -------------- | -------------------------------- | --------------------------------- | ------------------------------------ | ------------------------------------------------------------------|
| `install_bases`| Composite key: country_code, year| N/A                               | install_base NOT NULL                | One-to-Many: Each country_code linked to many users               |
| `users`        | user_id                          | country_code references install_bases| N/A                               | One-to-Many: Each user can have multiple entries in user_activity |
| `titles`       | title_id                         | N/A                               | name UNIQUE, NOT NULL                | One-to-Many: Each title associated with multiple user_activity    |
| `user_activity`| Composite key: user_id, title_id, timestamp | user_id references users, title_id references titles | N/A | Many-to-One: Many activity records linked to a single user and title |

### 4.3 MAU Calculation Process
1. Filter user activity data from the last 30 days.
2. Join with `users` and `install_bases` tables for country-specific install base data.
3. Sample and extrapolate MAUs based on the scaling factor for each country.
4. Use SQL queries for data extraction, sampling, and scaling:
```sql
WITH recent_activity AS (
  SELECT ua.user_id, ua.title_id, ua.timestamp, t.name, u.country_code
  FROM user_activity AS ua
  JOIN titles t ON ua.title_id = t.title_id
  JOIN users u ON ua.user_id = u.user_id
  WHERE ua.timestamp >= NOW() - INTERVAL '30 days'
    AND ua.title_id != 'Offline'
)
-- Additional SQL steps to calculate MAU
-- Step 1: Join user_activity table to titles table, filtering by timestamp (last 30 days) and excluding 'Offline'
WITH recent_activity AS (
    SELECT ua.user_id, ua.title_id, ua.timestamp, t.name, u.country_code
    FROM user_activity AS ua
    JOIN titles t ON ua.title_id = t.title_id
    JOIN users u ON ua.user_id = u.user_id
    WHERE ua.timestamp >= NOW() - INTERVAL '30 days'
      AND ua.title_id != 'Offline'
),

-- Step 2: Join recent_activity with install_bases to get the install base for the country and year
activity_with_install_base AS (
    SELECT ra.user_id, ra.title_id, ra.name, ra.country_code, ib.install_base
    FROM recent_activity AS ra
    JOIN install_bases ib ON ra.country_code = ib.country_code
    WHERE ib.year = EXTRACT(YEAR FROM NOW()) -- assuming you want the current year's install base
),

-- Step 3: Calculate sample size per country and scaling factor
sample_sizes AS (
    SELECT country_code, COUNT(DISTINCT user_id) AS sample_size
    FROM activity_with_install_base
    GROUP BY country_code
),

-- Step 4: Calculate the scaling factor for each country
scaling_factors AS (
    SELECT ss.country_code, ib.install_base, ss.sample_size, 
           (ib.install_base::DECIMAL / ss.sample_size) AS scaling_factor
    FROM sample_sizes AS ss
    JOIN install_bases ib ON ss.country_code = ib.country_code
    WHERE ib.year = EXTRACT(YEAR FROM NOW())
),

-- Step 5: Calculate the MAU for each game in each country
mau_per_title AS (
    SELECT awib.country_code, awib.title_id, awib.name, COUNT(DISTINCT awib.user_id) AS sample_mau,
           sf.scaling_factor, 
           (COUNT(DISTINCT awib.user_id) * sf.scaling_factor) AS final_mau_estimate
    FROM activity_with_install_base AS awib
    JOIN scaling_factors sf ON awib.country_code = sf.country_code
    GROUP BY awib.country_code, awib.title_id, awib.name, sf.scaling_factor
)

-- Step 6: Calculate margin of error for the estimates
SELECT mt.country_code, mt.title_id, mt.name, mt.sample_mau, mt.final_mau_estimate,
       -- Margin of error calculation
       1.96 * SQRT((mt.sample_mau / sf.sample_size) * (1 - (mt.sample_mau / sf.sample_size)) / sf.sample_size) AS margin_of_error
FROM mau_per_title AS mt
JOIN scaling_factors sf ON mt.country_code = sf.country_code
ORDER BY mt.country_code, mt.title_id;

### 4. Error Margin Calculation
A margin of error is calculated for each MAU estimate based on the sample size, confidence level, and population size. This provided an error range, allowing for more accurate reporting.

### 4.5 Possible Improvements
Several improvements were suggested for future iterations of the project:
- **MCMAU Data**: Gathering data on the Minimum Consistent MAU (MCMAU) for each region and adjusting the scaling factor monthly.
- **Scaling Factor Adjustments**: Instead of using yearly install base data, monthly scaling factors based on MCMAU should be applied, as a full install base is unlikely to be active every month.

### 4.6 Sampling Strategy
When selecting a sampling method to estimate MAUs with limited data access, **stratified sampling** was chosen for its accuracy across demographics. 

#### Why Stratified Sampling?
- Stratified sampling ensures representativity across various factors like country, title, and genre.
- **Country**: Install base data was used to determine the sample size per country for more accurate estimates.
- **Genre**: Sampling by genre within each market ensured a balanced representation of game types across geographic regions.

#### Example of Stratified Sampling:
- **Country**: Market A has 30% of the platform's install base, so 30% of the sample is allocated to this market.
- **Genre**: Genre X is played by 20% of Market A users, so 20% of Market A’s sample is allocated to this genre.

Random sampling is applied within each stratum, and optional weighting may be introduced if data for certain combinations is sparse.

## 5. Anticipating Challenges

### 5.1 Query Performance Issues
As the dataset grows, query performance may degrade. To address this:
- **Partitioning**: Dynamic range partitioning by timestamp was recommended for the `user_activity` table to handle large datasets efficiently.
- **Caching and Archiving**: Frequently accessed results can be cached, and older data can be archived to reduce table size and improve performance.

### 5.2 Data Discrepancies and Client Feedback
At times, discrepancies may arise between system estimates and client-provided data. The following steps were suggested:
- **Collaborative Investigation**: Work with clients to understand their data sources and estimation assumptions to identify the causes of divergence.
- **Adjusting the Model**: Adjust the estimation model when necessary, based on new insights or market-specific factors.
- **Improved Communication**: Establish a structured process to address client feedback, explaining the investigation process and addressing concerns.

### 5.3 Ensuring Data Quality
To maintain data quality, the following steps were recommended:
- **Scheduled Data Retrieval**: Automated data retrieval at regular intervals ensures timely updates.
- **Automated Data Tests**: Automated tests were implemented to validate data integrity, check constraints, and enforce data quality standards.
- **Descriptive Statistics and Outlier Detection**: Descriptive statistics and methods like Z-scores and IQR were used to detect anomalies in the data.
- **Automated Alerts**: Alerts were set up to notify the team in case of data extraction errors, ensuring quick resolution.
- **Version Control and Documentation**: Datasets were versioned and changes were documented to ensure full traceability.

## 6. Conclusion and Key Takeaways

The successful completion of this project resulted in an accurate and scalable method for estimating Monthly Active Users (MAUs) for individual video game titles across a wide range of markets. By leveraging multiple API endpoints, internal install base data, and a robust sampling strategy, the project provided stakeholders with actionable insights into player behavior across various regions.

### Key Takeaways:
- **Scalable Data Pipeline**: A well-designed database model capable of handling large datasets and generating daily MAU estimates.
- **Sampling Strategy and Data Accuracy**: The use of stratified sampling ensured representative data, even with access limitations.
- **Query Optimization and Automation**: Optimizations like indexing, partitioning, and automation improved query performance and streamlined the reporting process.
- **Collaboration and Client Feedback**: Open communication with clients helped resolve data discrepancies and refine estimation models.

This project demonstrated the value of thoughtful data modeling, accurate sampling strategies, and continuous process improvement to ensure reliable and actionable data for stakeholders.
This project demonstrated the value of thoughtful data modeling, careful sampling strategies, and the importance of continuous process improvement. The ability to adapt to challenges, optimize for performance, and ensure data accuracy through automation ultimately provided a reliable solution that could support both business objectives and client needs.

