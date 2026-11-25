import { useEffect, useState, useCallback } from 'react';
import jsPDF from 'jspdf';
import html2canvas from 'html2canvas';
import logo from './assets/itc-logo.png';
import './App.css';

function App() {
  const FRONTEND_URL = 'https://squiggly-happier-handful.ngrok-free.dev';
  const BACKEND_URL = window.location.origin;

  const [studentId, setStudentId] = useState('');
  const [mobile, setMobile] = useState('');
  const [otp, setOtp] = useState('');
  const [student, setStudent] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');
  const [flowStep, setFlowStep] = useState('id_search');
  const [isQrScan, setIsQrScan] = useState(false);
  const [isMobileDevice, setIsMobileDevice] = useState(() => {
    if (typeof window === 'undefined') return false;
    return window.innerWidth <= 768 || /Android|iPhone|iPad|iPod|Mobile/i.test(navigator.userAgent);
  });

  useEffect(() => {
    const checkMobile = () => {
      setIsMobileDevice(
        window.innerWidth <= 768 || /Android|iPhone|iPad|iPod|Mobile/i.test(navigator.userAgent)
      );
    };
    checkMobile();
    window.addEventListener('resize', checkMobile);
    return () => window.removeEventListener('resize', checkMobile);
  }, []);

  const lookupById = useCallback(
    async (overrideId) => {
      const cleanId = String(overrideId ?? studentId).trim();
      if (!cleanId) {
        setError('Please enter Student ID');
        return;
      }

      setLoading(true);
      setError('');
      setStudent(null);

      try {
        const response = await fetch(`${BACKEND_URL}/api/lookup-student/`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'ngrok-skip-browser-warning': 'true',
          },
          body: JSON.stringify({ student_id: cleanId }),
        });

        const data = await response.json();

        if (!response.ok) {
          setFlowStep('mobile_input');
          setError(data.error || 'Student ID not found. Please verify with your Mobile Number.');
          return;
        }

        setStudent(data);
      } catch {
        setError('Unable to connect to the backend server. Please check if Django is running.');
      } finally {
        setLoading(false);
      }
    },
    [BACKEND_URL, studentId]
  );

  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    const qrFlag = params.get('qr') || params.get('from_qr') || params.get('scan');
    const referrerIsQr = typeof document !== 'undefined' && /qr/i.test(document.referrer || '');
    const scannedQr = qrFlag === '1' || qrFlag === 'true' || referrerIsQr;
    setIsQrScan(scannedQr);

    const paramId = (params.get('studentId') || params.get('StudentId') || params.get('id') || '').trim();
    if (paramId) {
      setStudentId(paramId);
      lookupById(paramId);
    }
  }, [lookupById]);

  const sendOtp = async () => {
    const cleanMobile = mobile.trim();
    if (!cleanMobile || cleanMobile.length < 10) {
      setError('Please enter a valid mobile number');
      return;
    }

    setLoading(true);
    setError('');

    try {
      const response = await fetch(`${BACKEND_URL}/api/send-otp/`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'ngrok-skip-browser-warning': 'true',
        },
        body: JSON.stringify({ mobile: cleanMobile }),
      });

      const data = await response.json();
      if (!response.ok) throw new Error(data.error || 'Failed to send OTP');

      setFlowStep('otp_input');
      setError('');
      alert(`OTP sent successfully! (Debug OTP: ${data.debug_otp})`);
    } catch (err) {
      setError(err.message || 'Mobile number not found in database');
    } finally {
      setLoading(false);
    }
  };

  const verifyOtp = async () => {
    const cleanOtp = otp.trim();
    if (!cleanOtp || cleanOtp.length !== 6) {
      setError('Please enter a valid 6-digit OTP');
      return;
    }

    setLoading(true);
    setError('');

    try {
      const response = await fetch(`${BACKEND_URL}/api/verify-otp/`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'ngrok-skip-browser-warning': 'true',
        },
        body: JSON.stringify({ mobile: mobile.trim(), otp: cleanOtp }),
      });

      const data = await response.json();
      if (!response.ok) throw new Error(data.error || 'Invalid OTP');

      setStudent(data);
      setError('');
    } catch (err) {
      setError(err.message || 'Verification failed');
    } finally {
      setLoading(false);
    }
  };

  const getScheduleUrl = () => {
    if (!student) return '';
    const resolvedCourseId = String(student.course_id || '').trim();
    if (resolvedCourseId) {
      return `http://lms.itcmogri.org:801/Admin/a_publicSec.aspx?StudentId=${encodeURIComponent(
        student.student_id
      )}&CourseId=${encodeURIComponent(resolvedCourseId)}`;
    }
    if (student.schedule_url) return student.schedule_url;
    return `http://lms.itcmogri.org:801/Admin/a_publicSec.aspx?StudentId=${encodeURIComponent(
      student.student_id
    )}`;
  };

  const openStudentSchedule = (e) => {
    if (e) e.preventDefault();
    const schedUrl = getScheduleUrl();
    if (!schedUrl) {
      setError('Schedule link is not available for this student.');
      return;
    }
    setError('');
    window.location.assign(schedUrl);
  };

  const getCertificateFileUrl = () => {
    const rawUrl = (student?.certificate_url || '').trim();
    if (!rawUrl) return '';
    if (/^https?:\/\//i.test(rawUrl)) return rawUrl;
    const baseUrl = 'https://lms.itcmogri.org:801/StudentCertificate';
    const cleanPath = rawUrl.replace(/^\/+/, '').replace(/^StudentCertificate\/+/i, '');
    return `${baseUrl}/${cleanPath}`;
  };

  const downloadCertificate = async () => {
    const certUrl = getCertificateFileUrl();
    if (certUrl) {
      window.open(certUrl, '_blank', 'noopener,noreferrer');
      return;
    }

    const element = document.getElementById('certificate');
    if (!element || !student) return;

    const canvas = await html2canvas(element, {
      scale: 2,
      useCORS: true,
    });

    const imgData = canvas.toDataURL('image/png');
    const pdf = new jsPDF('landscape', 'mm', 'a4');
    const imgWidth = 297;
    const imgHeight = (canvas.height * imgWidth) / canvas.width;

    pdf.addImage(imgData, 'PNG', 0, 0, imgWidth, imgHeight);
    pdf.save(`${student.student_id}_Certificate.pdf`);
  };

  const qrTargetUrl = `${FRONTEND_URL}/?qr=1`;

  return (
    <div className="container">
      <div className="card">
        <div className="logo-section">
          <img src={logo} alt="ITC Logo" className="logo" />
          <h1>Information Technology Centre</h1>
          <p>{isQrScan ? 'Student Schedule Lookup' : 'Secure Certificate Verification'}</p>
        </div>

        {!student && (
          <>
            {flowStep === 'id_search' && (
              <>
                <div className="form-group">
                  <label>Enter Student ID</label>
                  <input
                    type="text"
                    placeholder="ENTER ID"
                    value={studentId}
                    onChange={(e) => setStudentId(e.target.value)}
                    onKeyDown={(e) => e.key === 'Enter' && lookupById()}
                  />
                </div>

                <button className="btn-primary" onClick={() => lookupById()} disabled={loading}>
                  {loading ? 'Searching...' : 'Lookup Student'}
                </button>
              </>
            )}

            {flowStep === 'mobile_input' && (
              <>
                <div className="form-group">
                  <label>Enter Mobile Number</label>
                  <input
                    type="text"
                    placeholder="Enter registered mobile"
                    value={mobile}
                    onChange={(e) => setMobile(e.target.value)}
                    onKeyDown={(e) => e.key === 'Enter' && sendOtp()}
                  />
                </div>

                <button className="btn-primary" onClick={sendOtp} disabled={loading}>
                  {loading ? 'Sending OTP...' : 'Send OTP'}
                </button>

                <button
                  className="btn-secondary"
                  style={{ marginTop: '10px' }}
                  onClick={() => {
                    setFlowStep('id_search');
                    setError('');
                  }}
                >
                  Back to ID Search
                </button>
              </>
            )}

            {flowStep === 'otp_input' && (
              <>
                <div className="form-group">
                  <label>Enter 6-Digit OTP sent to {mobile}</label>
                  <input
                    type="text"
                    placeholder="Enter or paste OTP"
                    maxLength="6"
                    value={otp}
                    onChange={(e) => setOtp(e.target.value)}
                    onKeyDown={(e) => e.key === 'Enter' && verifyOtp()}
                  />
                </div>

                <button className="btn-primary" onClick={verifyOtp} disabled={loading}>
                  {loading ? 'Verifying...' : 'Verify OTP'}
                </button>
              </>
            )}

            {error && <p className="error-text" style={{ marginTop: '15px' }}>{error}</p>}
          </>
        )}

        {student && (
          <div className="student-details">
            <h2>Student Details</h2>

            <table className="details-table">
              <tbody>
                <tr>
                  <td>Student ID</td>
                  <td><strong>{student.student_id}</strong></td>
                </tr>
                <tr>
                  <td>Student Name</td>
                  <td>{student.name}</td>
                </tr>
                <tr>
                  <td>Mobile No</td>
                  <td>{student.mobile || '-'}</td>
                </tr>
              </tbody>
            </table>

            {error && <p className="error-text" style={{ marginBottom: '12px' }}>{error}</p>}

            <div className="action-buttons" style={{ marginTop: '20px' }}>
              {isQrScan ? (
                isMobileDevice ? (
                  <a
                    href={getScheduleUrl()}
                    className="btn-primary"
                    style={{ display: 'block', textDecoration: 'none' }}
                    onClick={openStudentSchedule}
                  >
                    Student Schedule
                  </a>
                ) : (
                  <p className="error-text">
                    Student Schedule is available only on mobile devices.
                  </p>
                )
              ) : (
                <>
                  {isMobileDevice && (
                    <a
                      href={getScheduleUrl()}
                      className="btn-primary"
                      style={{ display: 'block', textDecoration: 'none' }}
                      onClick={openStudentSchedule}
                    >
                      Student Schedule
                    </a>
                  )}

                  <button className="btn-primary" onClick={downloadCertificate}>
                    Download Certificate
                  </button>

                  <button
                    className="btn-secondary"
                    onClick={() => {
                      setStudent(null);
                      setStudentId('');
                      setMobile('');
                      setOtp('');
                      setError('');
                      setFlowStep('id_search');
                    }}
                  >
                    Search Another
                  </button>
                </>
              )}
            </div>
          </div>
        )}

        {!student && !isQrScan && (
          <div style={{ textAlign: 'center', marginTop: '30px', borderTop: '1px solid #eee', paddingTop: '20px' }}>
            <img
              src={`https://api.qrserver.com/v1/create-qr-code/?size=150x150&data=${encodeURIComponent(qrTargetUrl)}`}
              alt="Portal QR Code"
              style={{ width: '130px', height: '130px', border: '4px solid white', boxShadow: '0 2px 8px rgba(0,0,0,0.1)' }}
            />
          </div>
        )}
      </div>

      {!isQrScan && student && (
        <div
          id="certificate"
          style={{
            width: '1123px',
            height: '794px',
            padding: '40px',
            background: 'white',
            position: 'absolute',
            left: '-9999px',
            top: 0,
            fontFamily: 'Georgia, serif',
            border: '14px solid #1e3a8a',
            boxSizing: 'border-box',
          }}
        >
          <div
            style={{
              border: '5px solid #c0a060',
              height: '100%',
              padding: '40px',
              textAlign: 'center',
              boxSizing: 'border-box',
            }}
          >
            <img src={logo} alt="ITC" style={{ height: '75px', marginBottom: '10px' }} />

            <h1 style={{ color: '#1e3a8a', fontSize: '34px', margin: '8px 0' }}>
              INFORMATION TECHNOLOGY CENTER
            </h1>

            <h2 style={{ color: '#b45309', fontSize: '26px', margin: '18px 0' }}>
              CERTIFICATE OF COMPLETION
            </h2>

            <p style={{ fontSize: '18px', margin: '20px 0' }}>
              This is to certify that
            </p>

            <h2
              style={{
                fontSize: '30px',
                color: '#1e3a8a',
                margin: '12px 0',
                textTransform: 'uppercase',
              }}
            >
              {student.name}
            </h2>

            <p style={{ fontSize: '17px', marginTop: '24px' }}>
              <strong>Student ID:</strong> {student.student_id}
            </p>
          </div>
        </div>
      )}
    </div>
  );
}

export default App;
