# Database Modelling Project

## 1. Project Overview

For this project, I was tasked with developing a methodology to estimate **Monthly Active Users (MAUs)** for individual video game titles across various global markets. This includes defining access rates, identifying what data points to extract from multiple API endpoints, and combining this data with internal sources to create accurate and scalable MAU estimates.

The challenge was to combine user activity data, title metadata, and platform install base statistics while accounting for privacy limitations and sampling biases. The overarching goal was to produce daily MAU estimates to support a wide range of reporting and analytical needs, including trend analysis and market comparisons.

My role involved:
- Planning the data extraction,
- Database design,
- Estimation methodology.

## 2. Challenges and Requirements

### Key Challenges:
- **API Limitations & Privacy Restrictions**: Data was available for only 1 in 4 users, due to privacy settings, making it difficult to capture a fully representative sample. This required careful planning of the sampling strategy and the frequency of data retrieval to maximize data coverage.
- **Time Zone Differences**: With gaming activity happening at different times in various countries, retrieving data at appropriate intervals was crucial. For accurate MAU estimates, the solution involved making multiple daily data requests, adjusted for different markets’ peak activity times.
- **Large-Scale Data Handling**: The database needed to process data for thousands of titles across dozens of markets. Scalability was essential to ensure that data retrieval, storage, and processing were efficient, while the results remained accurate.
- **Sampling Biases**: Given that not all users opted to share their activity data, developing a robust sampling strategy was critical. It involved applying stratified sampling techniques to ensure representative data collection across both geographic regions and genres.
- **Data Discrepancies**: At times, the MAU estimates produced by the system will differ from factual market data. It was important to respond to client concerns, reconcile differences, and adjust the estimation models as needed.

### Project Requirements:
- Modelling a database that could support real-time updates for MAU estimates..
- Ensuring the sampling strategy accounted for privacy restrictions and geographic diversity.
- Considering automated checks and testing to ensure data quality and accuracy before sharing results with clients.

## 3. Data Sources

The project needed to utilzie several API endpoints, alongside internal data sources, to estimate Monthly Active Users (MAUs) for individual game titles. Each data source played a crucial role in building a comprehensive understanding of user activity. The main data sources included:

- **User Activity Endpoint**: Provided information about what titles users were playing and when they were active. This endpoint returned a title_id if a user was currently playing a game or “Offline” if they were not. Due to privacy settings, this data was only available for about 25% of users, requiring careful sampling to ensure accurate estimations.
- **User Metadata Endpoint**: Contained user-specific data such as user_id, country_code, and creation_date. This was used to identify users’ countries and ensure accurate representation of different markets. The metadata was retrieved daily to account for new users joining the platform.
- **Title Metadata Endpoint**: Featured information on video game titles, such as title_id, name, genre, and release_date. This endpoint was essential for mapping user activity data to specific titles, and it helped categorize and filter games for MAU calculations.
- **Install Base Data**: : This internally available data provided yearly estimates of the active user base for each market, which was critical for scaling the sampled MAUs to population-wide estimates.

## 4. Approach and Solution
The approach to estimating Monthly Active Users (MAUs) for individual titles and markets involved several key phases, ranging from planning data extraction to the final calculation of estimates. Below is a breakdown of the process:

### 4.1 Data Extraction
The first step was to define which data to extract when from the available API endpoints. This involved the following actions:
- **User Activity Endpoint**: Data should be retrieved every 30 minutes to capture gaming sessions across different time zones and ensure adequate coverage of user activity. The extracted data included user_id and title_id, indicating which games users were playing at specific times.
- **User Metadata**: This data should be retrieved once a day to account for new users joining the platform. It included user_id and country_code, linking users to specific geographic markets.
- **Title Metadata Endpoints**: By pulling data daily, this endpoint provides information on new game releases and ensures that MAU estimates are linked to correct and up-to-date title information.
- **Internal Install Base Data**: Used to extrapolate the sampled MAUs to the population level for each country. The install base data provides the overall active user estimates for each market.
 
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
```
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

