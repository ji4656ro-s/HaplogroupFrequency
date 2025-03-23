# Haplogroup Frequency Explorer
Haplogroup Frequency Explorer is an interactive Streamlit application that enables users to visualize the global distribution of mitochondrial haplogroups. 

## Summary
- Load frequency data of haplogroups per country.
- Explore and visualize these frequencies on a color-coded heatmap.
- Generate a summary report highlighting the percentage of each haplogroup by country.

## Usage
1. Place Your Data Files
  - Haplogroup_Frequency_Per_Country.csv
  - ne_110m_admin_0_countries.shp plus all associated shapefile components (.shx, .dbf, .prj) in the same directory as the app or update the file paths in the script.
  - Run the App
    ```bash
    streamlit run app.py
    ```
  - Interact with the UI
      - Select a haplogroup from the dropdown.
      - Choose "View Heatmap" to see the choropleth map.
      - Or select "View Summary Report" for a detailed table.
## Data Requirements
- CSV File: Must contain 3 columns in this order:
   - Country
   - Haplogroup
   - Frequency

- Shapefile: A valid Natural Earth shapefile with country boundaries. The code expects a column named ADMIN that matches the CSV’s Country names.

## Dependencies
- Streamlit
- Pandas
- GeoPandas
- Matplotlib

## Python Script for the App

```python
import streamlit as st
import pandas as pd
import geopandas as gpd
import matplotlib.pyplot as plt
from matplotlib.colors import LinearSegmentedColormap

#Custom  for dark theme
st.markdown(
    """
    <style>
        /* Make entire page black with white text */
        body, .stApp {
            background-color: black !important;
            color: white !important;
        }

        /* Style the radio buttons (heatmap/summary) */
        div[data-baseweb="radio"] * {
            color: white !important;
        }

        /* Style the summary table */
        .stDataFrame {
            background-color: black !important;
            color: white !important;
        }

        /* Style input fields */
        input, textarea {
            background-color: black !important;
            color: white !important;
            border: 1px solid white !important;
        }

        /* Style buttons */
        .stButton>button {
            background-color: black !important;
            color: white !important;
            border: 1px solid white !important;
        }
    </style>
    """,
    unsafe_allow_html=True
)

#Load dataset with error handling
file_path = "Haplogroup_Frequency_Per_Country.csv"

try:
    df = pd.read_csv(file_path, delimiter=",", encoding="utf-8")
except FileNotFoundError:
    st.error(f"CSV file not found at: {file_path}")
    st.stop()
except Exception as e:
    st.error(f"Unexpected error reading CSV file:\n{e}")
    st.stop()

#Check if the CSV has the 3 columns as expected
required_columns = ["Country", "Haplogroup", "Frequency"]
if len(df.columns) < 3:
    st.error(f"CSV file does not have at least 3 columns. Found columns: {list(df.columns)}")
    st.stop()

# Rename columns safely, but check if they exist
df = df.iloc[:, [0, 1, 2]]  # Hard-coded indexing
df.columns = ["Country", "Haplogroup", "Frequency"]

# Validate that these columns exist after the rename
if not all(col in df.columns for col in required_columns):
    st.error(f"CSV is missing required columns. Found: {list(df.columns)}\nNeeded: {required_columns}")
    st.stop()

#Convert Frequency to numeric and validate data
df["Frequency"] = pd.to_numeric(df["Frequency"], errors="coerce")

# Check if everything turned NaN
if df["Frequency"].isna().all():
    st.error("All 'Frequency' values are invalid. Please check the CSV data.")
    st.stop()
elif df["Frequency"].isna().any():
    st.warning("Some 'Frequency' values are invalid and will be dropped.")
    df.dropna(subset=["Frequency"], inplace=True)

# Load world shapefile with error handling
shapefile_path="ne_110m_admin_0_countries.shp"
try:
    world = gpd.read_file(shapefile_path)
except FileNotFoundError:
    st.error(f"Shapefile not found at: {shapefile_path}")
    st.stop()
except Exception as e:
    st.error(f"Unexpected error reading shapefile:\n{e}")
    st.stop()

if world.empty:
    st.error("Shapefile is empty. Please verify it contains country geometry.")
    st.stop()

# Run Streamlit app
st.title(" Haplogroup Frequency Explorer")
st.write("Choose a haplogroup to visualize its global distribution.")

# Create a sorted list of unique haplogroups for selection
available_haplogroups = sorted(df["Haplogroup"].unique())
haplogroup_to_analyze = st.selectbox("Select Haplogroup:", available_haplogroups)

if haplogroup_to_analyze:
    # Filter dataset for the selected haplogroup
    df_filtered = df[df["Haplogroup"] == haplogroup_to_analyze]

    if df_filtered.empty:
        st.warning(f"No data found for haplogroup '{haplogroup_to_analyze}'. Try another one.")
        st.stop()
    
    # Compute summary statistics
    summary_df = df_filtered.groupby("Country", as_index=False).agg(
        Individuals_With_Haplogroup=("Frequency", "sum")
    )
    summary_df["Haplogroup_Frequency"] = (
        summary_df["Individuals_With_Haplogroup"] / summary_df["Individuals_With_Haplogroup"].sum() * 100
    )

    # Format the frequency column
    summary_df["Haplogroup_Frequency"] = summary_df["Haplogroup_Frequency"].apply(
        lambda x: "< 1%" if x < 1 else f"{x:.2f}%"
    )

    # Sort by highest to lowest incidence
    summary_df.sort_values(by="Individuals_With_Haplogroup", ascending=False, inplace=True)

    # Merge with world map
    world_haplogroup = world.merge(df_filtered, left_on="ADMIN", right_on="Country", how="left")

    # Option for the user to select which view to display
    option = st.radio("Select an option:", ["View Heatmap", "View Summary Report"])

    if option == "View Heatmap":
        st.subheader("Haplogroup Distribution Map")
        
        # Create the heatmap with black borders
        fig, ax = plt.subplots(figsize=(12, 6))

        # Plot country boundaries in black
        world.boundary.plot(ax=ax, linewidth=1.5, color="black")

        # Define custom colormap: light red to dark red
        custom_reds = LinearSegmentedColormap.from_list(
            "custom_reds",
            ["#ffcccc", "#ff6666", "#cc0000", "#660000"]  # light red → medium red → dark red
        )

        # Plot the heatmap with country edges in black
        # missing_kwds sets color for countries with no data
        world_haplogroup.plot(
            column="Frequency",
            cmap=custom_reds,
            legend=True,
            ax=ax,
            edgecolor="black",
            linewidth=0.8,
            missing_kwds={"color": "lightgrey"}
        )

        # Title in black text
        plt.title(
            f"Heatmap of {haplogroup_to_analyze} Distribution by Country",
            fontsize=14,
            color="black"
        )

        # Display the updated heatmap
        st.pyplot(fig)

    elif option == "View Summary Report":
        st.subheader("Summary Report")
        st.dataframe(
            summary_df.style.set_properties(
                **{'background-color': 'black', 'color': 'white', 'border-color': 'white'}
            )
        )
```
