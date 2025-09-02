import React, { useState, useEffect } from 'react';
import { submitLeave, getLeaveHistory, getEmployeeDetails } from '../services/api';
import { useParams } from 'react-router-dom';

const EmployeeDashboard = () => {
    const { empId } = useParams();

    const [fromDate, setFromDate] = useState('');
    const [toDate, setToDate] = useState('');
    const [leaveType, setLeaveType] = useState('CASUAL');
    const [remarks, setRemarks] = useState('');
    const [totalDays, setTotalDays] = useState(0);
    const [history, setHistory] = useState([]);
    const [managerId, setManagerId] = useState('');
    const [loading, setLoading] = useState(true);
    const [submitting, setSubmitting] = useState(false);
    const [error, setError] = useState('');

    // Calculate total days whenever dates change
    useEffect(() => {
        if (fromDate && toDate) {
            const diff = (new Date(toDate) - new Date(fromDate)) / (1000 * 60 * 60 * 24) + 1;
            setTotalDays(diff > 0 ? diff : 0);
        } else {
            setTotalDays(0);
        }
    }, [fromDate, toDate]);

    // Fetch employee details and leave history on mount
    useEffect(() => {
        const fetchData = async () => {
            setLoading(true);
            try {
                const empRes = await getEmployeeDetails(empId);
                setManagerId(empRes.data.managerId || '');

                const historyRes = await getLeaveHistory(empId);
                setHistory(historyRes.data);
            } catch (err) {
                console.error(err);
                setError('Failed to load data. Try refreshing.');
            }
            setLoading(false);
        };

        fetchData();
    }, [empId]);

    const handleSubmit = async (e) => {
        e.preventDefault();
        setError(''); // Reset error message before submitting

        if (!managerId) {
            setError("Manager not found. Cannot apply leave.");
            return;
        }

        if (totalDays <= 0) {
            setError("Invalid date range.");
            return;
        }

        const leavePayload = {
            fromDate,
            toDate,
            leaveType,
            remarks,
            numberOfDays: totalDays
        };

        // Log the leave payload to make sure it's being sent correctly
        console.log("Submit Leave Request", leavePayload);

        try {
            setSubmitting(true);

            // Send leave application to API
            await submitLeave(leavePayload, empId, managerId);  // pass empId and managerId here!

            // Log success message
            console.log("Leave applied successfully!");

            // Refresh leave history after submitting leave
            const historyRes = await getLeaveHistory(empId);
            setHistory(historyRes.data);

            // Reset form
            setFromDate('');
            setToDate('');
            setRemarks('');
            setTotalDays(0);
            setLeaveType('CASUAL');  // Reset the leave type to Casual after submission

        } catch (err) {
            console.error("Error applying leave:", err);
            setError('Error applying leave. Please check the form and try again.');
        } finally {
            setSubmitting(false);
        }
    };

    return (
        <div className="container mt-5">
            <h3 className="mb-4">Apply Leave</h3>
            {error && <div className="alert alert-danger">{error}</div>}

            <div className="card p-4 mb-5 shadow-sm">
                <form onSubmit={handleSubmit}>
                    <div className="row mb-3">
                        <div className="col">
                            <label>From Date</label>
                            <input
                                type="date"
                                className="form-control"
                                value={fromDate}
                                onChange={(e) => setFromDate(e.target.value)}
                                required
                            />
                        </div>
                        <div className="col">
                            <label>To Date</label>
                            <input
                                type="date"
                                className="form-control"
                                value={toDate}
                                onChange={(e) => setToDate(e.target.value)}
                                required
                            />
                        </div>
                    </div>

                    <div className="mb-3">
                        <label>Leave Type</label>
                        <select
                            className="form-select"
                            value={leaveType}
                            onChange={(e) => setLeaveType(e.target.value)}
                        >
                            <option value="CASUAL">Casual</option>
                            <option value="MEDICAL">Medical</option>
                            {/* Add more leave types here if needed */}
                        </select>
                    </div>

                    <div className="mb-3">
                        <label>Total Days</label>
                        <input type="number" className="form-control" value={totalDays} readOnly />
                    </div>

                    <div className="mb-3">
                        <label>Remarks</label>
                        <textarea
                            className="form-control"
                            value={remarks}
                            onChange={(e) => setRemarks(e.target.value)}
                            rows="3"
                        ></textarea>
                    </div>

                    <button type="submit" className="btn btn-success" disabled={submitting}>
                        {submitting ? 'Submitting...' : 'Submit'}
                    </button>
                </form>
            </div>

            <h4>Previous Leaves</h4>
            {history.length === 0 ? (
                <div>No leave history found.</div>
            ) : (
                <table className="table table-bordered">
                    <thead>
                        <tr>
                            <th>From</th>
                            <th>To</th>
                            <th>Type</th>
                            <th>Days</th>
                            <th>Remarks</th>
                            <th>Status</th>
                        </tr>
                    </thead>
                    <tbody>
                        {history.map((leave) => (
                            <tr key={leave.id}>
                                <td>{leave.fromDate}</td>
                                <td>{leave.toDate}</td>
                                <td>{leave.leaveType}</td>
                                <td>{leave.numberOfDays}</td>
                                <td>{leave.remarks}</td>
                                <td>{leave.leaveStatus}</td>
                            </tr>
                        ))}
                    </tbody>
                </table>
            )}
        </div>
    );
};

export default EmployeeDashboard;
