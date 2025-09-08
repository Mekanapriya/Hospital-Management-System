# Hospital-Management-System
import React, { useEffect, useState } from "react";

const API = "http://127.0.0.1:5000";

function App() {
  const [patients, setPatients] = useState([]);
  const [doctors, setDoctors] = useState([]);
  const [appointments, setAppointments] = useState([]);
  const [patientForm, setPatientForm] = useState({ name: "", phone: "", dob: "", address: "" });
  const [apptForm, setApptForm] = useState({ patient_id: "", doctor_id: "", scheduled_time: "", reason: "" });

  useEffect(() => {
    fetchPatients();
    fetchDoctors();
    fetchAppointments();
  }, []);

  async function fetchPatients() {
    const res = await fetch(`${API}/patients`);
    setPatients(await res.json());
  }
  async function fetchDoctors() {
    const res = await fetch(`${API}/doctors`);
    setDoctors(await res.json());
  }
  async function fetchAppointments() {
    const res = await fetch(`${API}/appointments`);
    setAppointments(await res.json());
  }

  async function submitPatient(e) {
    e.preventDefault();
    const res = await fetch(`${API}/patients`, {
      method: "POST",
      headers: {"Content-Type":"application/json"},
      body: JSON.stringify(patientForm)
    });
    if(res.ok){ setPatientForm({ name: "", phone: "", dob: "", address: "" }); fetchPatients(); }
    else alert('Error creating patient');
  }

  async function submitAppointment(e){
    e.preventDefault();
    // convert scheduled_time to ISO if needed
    const body = { ...apptForm, scheduled_time: apptForm.scheduled_time };
    const res = await fetch(`${API}/appointments`, {
      method: "POST",
      headers: {"Content-Type":"application/json"},
      body: JSON.stringify(body)
    });
    if(res.ok){ setApptForm({ patient_id: "", doctor_id: "", scheduled_time: "", reason: ""}); fetchAppointments(); }
    else { const txt = await res.text(); alert("Error creating appointment: " + txt); }
  }

  return (
    <div style={{padding:20, fontFamily:'Arial, sans-serif'}}>
      <h1>Hospital Management (Minimal)</h1>

      <section style={{display:'flex', gap:40}}>
        <div style={{flex:1}}>
          <h2>Patients</h2>
          <form onSubmit={submitPatient} style={{marginBottom:12}}>
            <input placeholder="Name" required value={patientForm.name} onChange={e=>setPatientForm({...patientForm, name:e.target.value})} />
            <input placeholder="Phone" value={patientForm.phone} onChange={e=>setPatientForm({...patientForm, phone:e.target.value})} />
            <input type="date" placeholder="DOB" value={patientForm.dob} onChange={e=>setPatientForm({...patientForm, dob:e.target.value})} />
            <input placeholder="Address" value={patientForm.address} onChange={e=>setPatientForm({...patientForm, address:e.target.value})} />
            <button type="submit">Add Patient</button>
          </form>
          <ul>
            {patients.map(p => <li key={p.id}>{p.name} — {p.phone} — {p.address}</li>)}
          </ul>
        </div>

        <div style={{flex:1}}>
          <h2>Doctors</h2>
          <ul>
            {doctors.map(d => <li key={d.id}>{d.name} — {d.specialization || 'General'}</li>)}
          </ul>

          <h3>New Appointment</h3>
          <form onSubmit={submitAppointment}>
            <select required value={apptForm.patient_id} onChange={e=>setApptForm({...apptForm, patient_id: Number(e.target.value)})}>
              <option value="">Select patient</option>
              {patients.map(p=> <option key={p.id} value={p.id}>{p.name}</option>)}
            </select>

            <select required value={apptForm.doctor_id} onChange={e=>setApptForm({...apptForm, doctor_id: Number(e.target.value)})}>
              <option value="">Select doctor</option>
              {doctors.map(d=> <option key={d.id} value={d.id}>{d.name} ({d.specialization})</option>)}
            </select>

            <input type="datetime-local" required value={apptForm.scheduled_time} onChange={e=>setApptForm({...apptForm, scheduled_time: e.target.value})}/>
            <input placeholder="Reason" value={apptForm.reason} onChange={e=>setApptForm({...apptForm, reason: e.target.value})} />
            <button type="submit">Schedule</button>
          </form>
        </div>
      </section>

      <section style={{marginTop:30}}>
        <h2>Appointments</h2>
        <table border="1" cellPadding="8" style={{borderCollapse:'collapse', width:'100%'}}>
          <thead>
            <tr>
              <th>ID</th><th>Patient</th><th>Doctor</th><th>When</th><th>Reason</th><th>Status</th>
            </tr>
          </thead>
          <tbody>
            {appointments.map(a => (
              <tr key={a.id}>
                <td>{a.id}</td>
                <td>{a.patient_id}</td>
                <td>{a.doctor_id}</td>
                <td>{new Date(a.scheduled_time).toLocaleString()}</td>
                <td>{a.reason}</td>
                <td>{a.status}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>
    </div>
  );
}

export default App;
Flask==2.2.5
Flask-Cors==3.0.10
Flask-SQLAlchemy==3.0.3
marshmallow==3.20.1
app.py
python
Copy code
from flask import Flask, request, jsonify, abort
from flask_sqlalchemy import SQLAlchemy
from flask_cors import CORS
from datetime import datetime
from marshmallow import Schema, fields

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///hms.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)
CORS(app)

# --- Models ---
class Doctor(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(120), nullable=False)
    specialization = db.Column(db.String(120))
    phone = db.Column(db.String(50))

class Patient(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(120), nullable=False)
    dob = db.Column(db.Date)
    phone = db.Column(db.String(50))
    address = db.Column(db.String(255))

class Appointment(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    patient_id = db.Column(db.Integer, db.ForeignKey('patient.id'), nullable=False)
    doctor_id = db.Column(db.Integer, db.ForeignKey('doctor.id'), nullable=False)
    scheduled_time = db.Column(db.DateTime, nullable=False)
    reason = db.Column(db.String(255))
    status = db.Column(db.String(50), default='scheduled')

    patient = db.relationship('Patient', backref=db.backref('appointments', lazy=True))
    doctor = db.relationship('Doctor', backref=db.backref('appointments', lazy=True))

class Prescription(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    appointment_id = db.Column(db.Integer, db.ForeignKey('appointment.id'), nullable=False)
    content = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    appointment = db.relationship('Appointment', backref=db.backref('prescriptions', lazy=True))

# --- Schemas (marshmallow) ---
class DoctorSchema(Schema):
    id = fields.Int(dump_only=True)
    name = fields.Str(required=True)
    specialization = fields.Str()
    phone = fields.Str()

class PatientSchema(Schema):
    id = fields.Int(dump_only=True)
    name = fields.Str(required=True)
    dob = fields.Date()
    phone = fields.Str()
    address = fields.Str()

class AppointmentSchema(Schema):
    id = fields.Int(dump_only=True)
    patient_id = fields.Int(required=True)
    doctor_id = fields.Int(required=True)
    scheduled_time = fields.DateTime(required=True)
    reason = fields.Str()
    status = fields.Str()

class PrescriptionSchema(Schema):
    id = fields.Int(dump_only=True)
    appointment_id = fields.Int(required=True)
    content = fields.Str(required=True)
    created_at = fields.DateTime(dump_only=True)

doctor_schema = DoctorSchema()
doctors_schema = DoctorSchema(many=True)
patient_schema = PatientSchema()
patients_schema = PatientSchema(many=True)
appointment_schema = AppointmentSchema()
appointments_schema = AppointmentSchema(many=True)
prescription_schema = PrescriptionSchema()
prescriptions_schema = PrescriptionSchema(many=True)

# --- Utility: init DB route (for dev only) ---
@app.route('/init-db', methods=['POST'])
def init_db():
    db.drop_all()
    db.create_all()
    # optional seed:
    d1 = Doctor(name='Dr. A. Singh', specialization='Cardiology', phone='9999999999')
    d2 = Doctor(name='Dr. R. Gupta', specialization='Pediatrics', phone='9888888888')
    p1 = Patient(name='John Doe', dob=datetime(1990,1,1), phone='7777777777', address='Mumbai')
    p2 = Patient(name='Jane Roe', dob=datetime(1985,5,20), phone='6666666666', address='Delhi')
    db.session.add_all([d1,d2,p1,p2])
    db.session.commit()
    return jsonify({'message':'db initialized'}), 201

# --- Doctor endpoints ---
@app.route('/doctors', methods=['GET'])
def list_doctors():
    doctors = Doctor.query.all()
    return jsonify(doctors_schema.dump(doctors))

@app.route('/doctors/<int:doc_id>', methods=['GET'])
def get_doctor(doc_id):
    doc = Doctor.query.get_or_404(doc_id)
    return doctor_schema.dump(doc)

@app.route('/doctors', methods=['POST'])
def create_doctor():
    data = request.get_json()
    err = doctor_schema.validate(data)
    if err:
        return jsonify(err), 400
    doc = Doctor(**data)
    db.session.add(doc)
    db.session.commit()
    return doctor_schema.dump(doc), 201

@app.route('/doctors/<int:doc_id>', methods=['PUT'])
def update_doctor(doc_id):
    doc = Doctor.query.get_or_404(doc_id)
    data = request.get_json()
    for k,v in data.items():
        setattr(doc, k, v)
    db.session.commit()
    return doctor_schema.dump(doc)

@app.route('/doctors/<int:doc_id>', methods=['DELETE'])
def delete_doctor(doc_id):
    doc = Doctor.query.get_or_404(doc_id)
    db.session.delete(doc)
    db.session.commit()
    return jsonify({'message':'deleted'})

# --- Patient endpoints ---
@app.route('/patients', methods=['GET'])
def list_patients():
    patients = Patient.query.all()
    return jsonify(patients_schema.dump(patients))

@app.route('/patients/<int:pid>', methods=['GET'])
def get_patient(pid):
    p = Patient.query.get_or_404(pid)
    return patient_schema.dump(p)

@app.route('/patients', methods=['POST'])
def create_patient():
    data = request.get_json()
    err = patient_schema.validate(data)
    if err:
        return jsonify(err), 400
    # allow dob as string or date
    if 'dob' in data and isinstance(data['dob'], str):
        try:
            data['dob'] = datetime.fromisoformat(data['dob']).date()
        except Exception:
            pass
    p = Patient(**data)
    db.session.add(p)
    db.session.commit()
    return patient_schema.dump(p), 201

@app.route('/patients/<int:pid>', methods=['PUT'])
def update_patient(pid):
    p = Patient.query.get_or_404(pid)
    data = request.get_json()
    for k,v in data.items():
        setattr(p, k, v)
    db.session.commit()
    return patient_schema.dump(p)

@app.route('/patients/<int:pid>', methods=['DELETE'])
def delete_patient(pid):
    p = Patient.query.get_or_404(pid)
    db.session.delete(p)
    db.session.commit()
    return jsonify({'message':'deleted'})

# --- Appointments ---
@app.route('/appointments', methods=['GET'])
def list_appointments():
    appts = Appointment.query.order_by(Appointment.scheduled_time.desc()).all()
    return jsonify(appointments_schema.dump(appts))

@app.route('/appointments/<int:aid>', methods=['GET'])
def get_appointment(aid):
    a = Appointment.query.get_or_404(aid)
    return appointment_schema.dump(a)

@app.route('/appointments', methods=['POST'])
def create_appointment():
    data = request.get_json()
    err = appointment_schema.validate(data)
    if err:
        return jsonify(err), 400
    # parse scheduled_time
    if isinstance(data.get('scheduled_time'), str):
        data['scheduled_time'] = datetime.fromisoformat(data['scheduled_time'])
    a = Appointment(**data)
    db.session.add(a)
    db.session.commit()
    return appointment_schema.dump(a), 201

@app.route('/appointments/<int:aid>', methods=['PUT'])
def update_appointment(aid):
    a = Appointment.query.get_or_404(aid)
    data = request.get_json()
    if 'scheduled_time' in data and isinstance(data['scheduled_time'], str):
        data['scheduled_time'] = datetime.fromisoformat(data['scheduled_time'])
    for k,v in data.items():
        setattr(a, k, v)
    db.session.commit()
    return appointment_schema.dump(a)

@app.route('/appointments/<int:aid>', methods=['DELETE'])
def delete_appointment(aid):
    a = Appointment.query.get_or_404(aid)
    db.session.delete(a)
    db.session.commit()
    return jsonify({'message':'deleted'})

# --- Prescriptions ---
@app.route('/prescriptions', methods=['GET'])
def list_prescriptions():
    pres = Prescription.query.all()
    return jsonify(prescriptions_schema.dump(pres))

@app.route('/prescriptions', methods=['POST'])
def create_prescription():
    data = request.get_json()
    err = prescription_schema.validate(data)
    if err:
        return jsonify(err), 400
    pres = Prescription(**data)
    db.session.add(pres)
    db.session.commit()
    return prescription_schema.dump(pres), 201

# --- Basic error handlers ---
@app.errorhandler(404)
def not_found(e):
    return jsonify({'error':'Not found'}), 404

if __name__ == '__main__':
    app.run(debug=True)
    

    
