# Assmnt
import pandas as pd
import json
import numpy as np
from datetime import datetime

# ------------------------------------------------------------
# 1. LOAD AND INTEGRATE THE DATA (ASSUMING CORRECT FILES)
# ------------------------------------------------------------
# Load orders.csv
orders = pd.read_csv('orders.csv', parse_dates=['order_date'], dayfirst=True)

# Load users.json
with open('users.json', 'r') as f:
    users_data = json.load(f)
users = pd.DataFrame(users_data)

# Load restaurants from the SQL file (parsing INSERT statements)
# NOTE: The provided restaurants.sql file is incomplete in the text.
# You MUST use the FULL, original file for this to work.
with open('restaurants.sql', 'r') as f:
    sql_content = f.read()

# Parse INSERT statements (simplified - assumes standard format)
import re
pattern = r'INSERT INTO restaurants VALUES\s*\((\d+),\s*\'([^\']+)\',\s*\'([^\']+)\',\s*([\d.]+)\);'
matches = re.findall(pattern, sql_content)

restaurants_data = []
for match in matches:
    restaurants_data.append({
        'restaurant_id': int(match[0]),
        'restaurant_name': match[1],
        'cuisine': match[2],
        'rating': float(match[3])
    })
restaurants = pd.DataFrame(restaurants_data)

# Perform the joins (LEFT JOIN on orders)
df = orders.merge(users, on='user_id', how='left')
df = df.merge(restaurants, on='restaurant_id', how='left', suffixes=('_order', '_restaurant'))

# ------------------------------------------------------------
# 2. ANSWER EACH MULTIPLE-CHOICE QUESTION
# ------------------------------------------------------------
answers = {}

# Q1: Which city has the highest total revenue from Gold members?
gold_rev = df[df['membership'] == 'Gold'].groupby('city')['total_amount'].sum()
answers['Q1'] = gold_rev.idxmax()

# Q2: Which cuisine has the highest average order value?
avg_aov_cuisine = df.groupby('cuisine')['total_amount'].mean()
answers['Q2'] = avg_aov_cuisine.idxmax()

# Q3: How many distinct users placed orders worth more than ₹1000 in total?
user_totals = df.groupby('user_id')['total_amount'].sum()
distinct_users_over_1000 = (user_totals > 1000).sum()
# Map count to option range
if distinct_users_over_1000 < 500:
    answers['Q3'] = '< 500'
elif 500 <= distinct_users_over_1000 <= 1000:
    answers['Q3'] = '500 – 1000'
elif 1000 < distinct_users_over_1000 <= 2000:
    answers['Q3'] = '1000 – 2000'
else:
    answers['Q3'] = '> 2000'

# Q4: Which restaurant rating range generated the highest total revenue?
bins = [3.0, 3.6, 4.1, 4.6, 5.01]  # 5.01 to include 5.0
labels = ['3.0 – 3.5', '3.6 – 4.0', '4.1 – 4.5', '4.6 – 5.0']
df['rating_range'] = pd.cut(df['rating'], bins=bins, labels=labels, right=False)
rev_by_rating = df.groupby('rating_range')['total_amount'].sum()
answers['Q4'] = rev_by_rating.idxmax()

# Q5: Among Gold members, which city has the highest average order value?
gold_avg_aov = df[df['membership'] == 'Gold'].groupby('city')['total_amount'].mean()
answers['Q5'] = gold_avg_aov.idxmax()

# Q6: Which cuisine has the lowest number of distinct restaurants but still contributes significant revenue?
# Define "significant revenue" as total revenue > (overall_mean_revenue_per_cuisine)
restaurant_count = df.groupby('cuisine')['restaurant_id'].nunique()
revenue_per_cuisine = df.groupby('cuisine')['total_amount'].sum()
# Filter cuisines with above-average revenue
significant_cuisines = revenue_per_cuisine[revenue_per_cuisine > revenue_per_cuisine.mean()]
filtered_restaurant_count = restaurant_count[significant_cuisines.index]
answers['Q6'] = filtered_restaurant_count.idxmin()

# Q7: What percentage of total orders were placed by Gold members? (Rounded)
total_orders = df['order_id'].nunique()
gold_orders = df[df['membership'] == 'Gold']['order_id'].nunique()
gold_percentage = round((gold_orders / total_orders) * 100)
# Map to nearest option
options_q7 = [40, 45, 50, 55]
answers['Q7'] = f"{min(options_q7, key=lambda x: abs(x - gold_percentage))}%"

# Q8: Which restaurant has the highest average order value but less than 20 total orders?
restaurant_stats = df.groupby('restaurant_name_order').agg(
    avg_order_value=('total_amount', 'mean'),
    order_count=('order_id', 'nunique')
)
filtered = restaurant_stats[restaurant_stats['order_count'] < 20]
if not filtered.empty:
    answers['Q8'] = filtered['avg_order_value'].idxmax()
else:
    answers['Q8'] = "None meet criteria"

# Q9: Which combination contributes the highest revenue?
df['combo'] = df['membership'] + ' + ' + df['cuisine'] + ' cuisine'
combo_revenue = df.groupby('combo')['total_amount'].sum()
answers['Q9'] = combo_revenue.idxmax()

# Q10: During which quarter is the total revenue highest?
df['quarter'] = df['order_date'].dt.quarter
quarter_map = {1: 'Q1 (Jan–Mar)', 2: 'Q2 (Apr–Jun)', 3: 'Q3 (Jul–Sep)', 4: 'Q4 (Oct–Dec)'}
quarter_rev = df.groupby('quarter')['total_amount'].sum()
answers['Q10'] = quarter_map[quarter_rev.idxmax()]

# ------------------------------------------------------------
# 3. PRINT THE ANSWERS
# ------------------------------------------------------------
print("ANSWERS TO MULTIPLE-CHOICE QUESTIONS")
print("="*45)
for q, a in answers.items():
    print(f"{q}: {a}")
