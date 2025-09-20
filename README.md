import React, { useState, useEffect } from 'react';
import { submitLeave, getLeaveHistory, getEmployeeDetails } from '../services/api';
import { useParams } from 'react-router-dom';
                           
const EmployeeDashboard = () => {
    const { empId } = useParams();
 
    const [fromDate, setFromDate] = useState('');
    const [toDate, setToDate] = useState('');
    const [leaveType, setLeaveType] = useState('CASUAL'); // Default leave type
    const [remarks, setRemarks] = useState('');
    const [totalDays, setTotalDays] = useState(0);
    const [history, setHistory] = useState([]);
    const [managerId, setManagerId] = useState('');
    const [employeeName, setEmployeeName] = useState(''); // New state for employee's name
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
                console.log('Employee Details:', empRes); // Log the response to check if the name is there
                setManagerId(empRes.data.managerId || '');
                
                // Concatenate firstName and lastName to set the employee's full name
                const fullName = `${empRes.data.firstName} ${empRes.data.lastName}`;
                setEmployeeName(fullName || ''); // Set the employee's full name

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
        setError('');

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

        try {
            setSubmitting(true);
            await submitLeave(leavePayload, empId, managerId);  // pass empId and managerId here!
            alert('Leave applied successfully!');
            
            // Refresh leave history
            const historyRes = await getLeaveHistory(empId);
            setHistory(historyRes.data);

            // Reset form (without resetting leaveType)
            setFromDate('');
            setToDate('');
            setRemarks('');
            setTotalDays(0);
            // leaveType should remain as the last selected type, so no need to reset it here.
        } catch (err) {
            console.error(err);
            setError('Error applying leave. Please check the form and try again.');
        } finally {
            setSubmitting(false);
        }
    };

    return (
        <div className="container mt-5">
            {/* Displaying the employee's full name at the top */}
            <h2>{employeeName ? `Welcome ${employeeName}!` : 'Loading...'}</h2> {/* Display employee name if available */}

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
                            <option value="Casual">Casual</option>
                            <option value="Medical">Medical</option>
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
