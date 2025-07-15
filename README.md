# msme2import streamlit as st
import pandas as pd

st.set_page_config(page_title="MSME Calculation Tool", layout="wide")

st.title("📊 MSME Calculation Web App")

# Step 1: Upload creditors Excel file
st.header("Step 1: Upload Creditors Excel File")
creditors_file = st.file_uploader("Upload Excel File", type=["xlsx"])

if creditors_file:
    df_creditors = pd.read_excel(creditors_file)
    st.subheader("📄 Uploaded Data")
    st.dataframe(df_creditors, use_container_width=True)  # Display as-is

    # Step 3: Calculation Eligibility
    st.header("Step 3: MSME Eligibility Calculation")
    df_creditors["Net"] = df_creditors["Debit"] - df_creditors["Credit"]
    eligible_df = df_creditors[df_creditors["Net"] > 0].copy()
    st.dataframe(eligible_df, use_container_width=True)

    # Step 4: Agreement Check
    st.header("Step 4: Agreement Check")
    agreement_days = []
    for index, row in eligible_df.iterrows():
        party_name = row["Name of party"]
        with st.expander(f"Agreement details for: {party_name}"):
            is_agreement = st.radio(f"Is there an agreement for {party_name}?", ("Yes", "No"), key=f"agree_{index}")
            if is_agreement == "Yes":
                days = st.number_input("Enter number of days in agreement:", min_value=1, max_value=45, key=f"days_{index}")
                agreement_days.append(min(days, 45))
            else:
                agreement_days.append(15)
    eligible_df["Agreement Days"] = agreement_days

    # Step 5: Upload 15 Feb – 31 Mar Summary
    st.header("Step 5: Upload 15 Feb - 31 Mar Summary File")
    feb_file = st.file_uploader("Upload Excel File for 15 Feb - 31 Mar", type=["xlsx"])
    if feb_file:
        df_feb = pd.read_excel(feb_file)
        st.subheader("📄 Uploaded 15 Feb - 31 Mar Data")
        st.dataframe(df_feb, use_container_width=True)

        # Filter only parties in eligible_df
        filtered_feb_df = df_feb[df_feb["Name of party"].isin(eligible_df["Name of party"])]
        st.subheader("Filtered Data for MSME Parties")
        st.dataframe(filtered_feb_df, use_container_width=True)

        # Step 6: Apply Closing Balance Logic
        allowed_list = []
        disallowed_list = []

        for index, row in filtered_feb_df.iterrows():
            cr_dr_diff = row["Credit"] - row["Debit"]
            closing_balance = row["Closing Balance"]

            if closing_balance >= cr_dr_diff:
                allowed = cr_dr_diff
                disallowed = closing_balance - cr_dr_diff
            else:
                allowed = closing_balance
                disallowed = 0

            allowed_list.append(allowed)
            disallowed_list.append(disallowed)

        filtered_feb_df["Allowed Amount"] = allowed_list
        filtered_feb_df["Disallowed Amount"] = disallowed_list

        st.subheader("✅ MSME Calculations Result")
        st.dataframe(filtered_feb_df, use_container_width=True)

        # Download final result
        output = filtered_feb_df.to_excel(index=False)
        st.download_button("📥 Download Final Excel", output, file_name="MSME_Calculations.xlsx", mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")

