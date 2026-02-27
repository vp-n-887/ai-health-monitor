import streamlit as st
import requests
import json

# --- Page Configurations ---
st.set_page_config(page_title="Telehealth AI Assistant", layout="wide")

# --- Custom N8N Webhook URLs (REPLACE THESE WITH PRODUCTION URLS) ---
N8N_WEBHOOK_DOCTOR_CONFIG = "https://am-space.app.n8n.cloud/webhook-test/DOCTORDATA"
N8N_WEBHOOK_GET_PATIENT_PARAMS = "https://your-n8n-instance.com/webhook/get-patient-params"
N8N_WEBHOOK_PROCESS_SUBMISSION = "https://your-n8n-instance.com/webhook/process-submission"

# --- Application Title ---
st.title("AI-Powered Patient Recovery Monitoring")

# --- Sidebar for Navigation ---
app_mode = st.sidebar.selectbox("Choose Interface", ["Doctor's Panel", "Patient's Portal"])

# ==========================================
# 1. Doctor's Interface (UPDATED)
# ==========================================
if app_mode == "Doctor's Panel":
    st.header("1. Doctor's Configuration Panel")
    st.subheader("Define monitoring parameters for a new patient.")

    with st.form("doctor_config_form"):
        col1, col2 = st.columns(2)
        with col1:
            # Labels and keys now match your n8n mapping exactly
            doc_name = st.text_input("Doc name")
            patient_name = st.text_input("Patient Name")
            patient_age = st.number_input("Patient Age", min_value=0, max_value=120, step=1)
        with col2:
            surgery_type = st.text_input("Surgery Type")
            monitoring_setup = st.radio("Monitoring Setup", ("AI-generated", "Manual"))

        submitted = st.form_submit_button("Configure and Save to Sheets")

    if submitted:
        if not all([doc_name, patient_name, surgery_type]):
            st.error("Please fill in the required fields.")
        else:
            # The keys here match your $json['Key Name'] in n8n
            data_to_send = {
                "Doc Name": doc_name,
                "Patient Name": patient_name,
                "Patient Age": patient_age,
                "Surgery Type": surgery_type,
                "Monitoring Setup": monitoring_setup
            }

            with st.spinner("Appending to Google Sheets via n8n..."):
                try:
                    response = requests.post(N8N_WEBHOOK_DOCTOR_CONFIG, json=data_to_send)
                    response.raise_for_status()
                    st.success(f"Successfully configured monitoring for {patient_name}!")
                except requests.exceptions.RequestException as e:
                    st.error(f"Error connecting to n8n: {e}")

# ==========================================
# 2. Patient's Interface
# ==========================================
elif app_mode == "Patient's Portal":
    st.header("2. Patient's Daily Portal")
    
    if "patient_params" not in st.session_state:
        with st.form("patient_login_form"):
            login_patient_name = st.text_input("Your Full Name to begin")
            login_submitted = st.form_submit_button("Fetch My Check-in Form")

        if login_submitted and login_patient_name:
            try:
                response = requests.post(N8N_WEBHOOK_GET_PATIENT_PARAMS, json={"patient_name": login_patient_name})
                response.raise_for_status()
                patient_params = response.json()

                if patient_params and isinstance(patient_params, list):
                    st.session_state["patient_params"] = patient_params
                    st.session_state["login_patient_name"] = login_patient_name
                    st.rerun() # Updated from experimental_rerun
                else:
                    st.warning("No monitoring plan found.")
            except Exception as e:
                st.error(f"Error: {e}")

    if "patient_params" in st.session_state:
        st.write(f"### Form for: **{st.session_state['login_patient_name']}**")
        input_data = {}
        files_to_send = {}

        for param in st.session_state["patient_params"]:
            p_name = param.get("name")
            p_type = param.get("data_type", "text").lower()
            if p_type == "text": input_data[p_name] = st.text_input(p_name)
            elif p_type == "number": input_data[p_name] = st.number_input(p_name)
            elif p_type == "audio":
                audio = st.file_uploader(f"Upload Audio for {p_name}", type=["mp3", "wav"])
                if audio: files_to_send["audio_file"] = (audio.name, audio, audio.type)
            elif p_type == "image":
                img = st.file_uploader(f"Upload Image for {p_name}", type=["jpg", "png"])
                if img: files_to_send["image_file"] = (img.name, img, img.type)

        if st.button("Submit My Daily Check-in"):
            payload = {"patient_name": st.session_state["login_patient_name"], "form_data": input_data}
            try:
                res = requests.post(N8N_WEBHOOK_PROCESS_SUBMISSION, data={"main_data": json.dumps(payload)}, files=files_to_send)
                res.raise_for_status()
                analysis = res.json()
                st.success("Analysis Complete!")
                st.write(f"**Status:** {analysis.get('recovery_status')}")
                st.progress(analysis.get("recovery_rate", 0) / 100)
            except Exception as e:
                st.error(f"Submission failed: {e}")

        if st.button("Clear Form"):
            st.session_state.clear()
            st.rerun()
