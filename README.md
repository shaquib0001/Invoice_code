# Invoice_code

import React, { useEffect, useRef, useState } from "react";
import { ComboboxItem, Divider, Flex, Group, Modal, OptionsFilter, Select, ScrollArea, Textarea } from "@mantine/core";
import {
  TextInput,
  Table,
  Loader,
  Center,
  Notification,
  Button,
  NumberInput,
  Checkbox,
} from "@mantine/core";
import axios from "axios";
import "./Invoice.css";
import { fetchLatestInvoice, fetchUpdateInvoiceStatus, getAllDuplicate, getAllInstructorDetails, getAllInstructorsDetail } from "../api/dynamodb-db-api";
import { useLocalStorage } from '@mantine/hooks';  // Adjust according to where your hook is located
import { User } from "../common/models";
import { Text } from '@mantine/core';  // Make sure this import is present



interface InvoiceData {
  inv_no: string; // Primary key
  bank_ifsc?: string;
  date?: string;
  trainerName?: string;
  courseName?: string;
  batchNumber1?: number;
  batchNumber2?: number;
  batchNumber3?: number;
  numberOfStudents?: number;
  studentNames?: string[];
  numOfClasses?: string[];
  leaves?: number;
  classes?: number;
  prorata?: boolean;
  numClassesAsMOU?: number;
  costAsPerMOU?: number;
  trainerAskedPayment?: number;
  tds?: number;
  netPayable?: number;
  paymentPercentage?: number;
  instructor_bank_account?: string;
  instructor_contact?: string;
  instructor_email?: string;
  instructor_pan?: string;
  instructor_upi_id?: string;
  leavesAmountDeducted?: number;
  status?: string;
  Email?: string;
  Contact?: string;
  mouHours?: string;
  tdsStatus?: string;

  customCalculation?: boolean; // For Custom Calculation Checkbox
  tdsAmount?: number;
}
//  interface UserProps {
//  user: User;
//  }

const InvoicePage: React.FC = () => {
  const [tableData, setTableData] = useState<InvoiceData[]>([]);
  const [filteredData, setFilteredData] = useState<InvoiceData[]>([]);
  const [loading, setLoading] = useState<boolean>(true);
  const [error, setError] = useState<string | null>(null);

  const [trainerNameFilter, setTrainerNameFilter] = useState<string>("");
  const [invNoFilter, setInvNoFilter] = useState<string>("");
  const [currentPage, setCurrentPage] = useState(1); // Current page
  const [rowsPerPage, setRowsPerPage] = useState(15); // Rows per page

  // Calculate the range of rows for the current page
  const indexOfLastRow = currentPage * rowsPerPage;
  const indexOfFirstRow = indexOfLastRow - rowsPerPage;
  const currentRows = filteredData.slice(indexOfFirstRow, indexOfLastRow);

  // Calculate total pages
  const totalPages = Math.ceil(filteredData.length / rowsPerPage);


  const [showForm, setShowForm] = useState<boolean>(false);


  const [formData, setFormData] = useState<InvoiceData>({
    inv_no: "",
    date: "",
    trainerName: "",
    courseName: "",
    batchNumber1: 0,
    batchNumber2: 0,
    batchNumber3: 0,
    numberOfStudents: 0,
    leaves: 0,
    classes: 0,
    prorata: false,
    numClassesAsMOU: 0,
    costAsPerMOU: 0,
    trainerAskedPayment: 0,
    tds: 0,
    netPayable: 0,
    status: "Open",
    Email: "",
    Contact: "",
    studentNames: [],
    numOfClasses: [],
    paymentPercentage: 0,
    instructor_bank_account: "",
    instructor_contact: "",
    instructor_email: "",
    instructor_pan: "",
    instructor_upi_id: "",
    leavesAmountDeducted: 0,
    mouHours: "",
    tdsStatus: "",
  });
  console.log(formData, "login Form Data")
  const [selectedInvoice, setSelectedInvoice] = useState<InvoiceData | null>(null);
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [isDuplicateModalOpen, setIsDuplicateModalOpen] = useState(false);

  const [invoices, setInvoices] = useState<InvoiceData[]>([]); // List of invoices
  const [duplicates, setDuplicates] = useState<{ batchNumber: string; ids: string[] }[]>([]);
  const [instructors, setInstructors] = useState<ComboboxItem[]>([]);
  const instructorsRef = useRef<instructors[]>([]);
  const [isEditing, setIsEditing] = useState(false);
  const [user] = useLocalStorage<User | undefined>({ key: 'user' });

  // console.log(user,"login User Data");
  // const [isDuplicatesModalOpen, setIsDuplicatesModalOpen] = useState(false);


  // Function to open the modal with selected invoice
  const handleInvoiceClick = (invoice: InvoiceData) => {
    console.log(invoice, "Invoice Data")

    setSelectedInvoice(invoice);
    // setIsDuplicateModalOpen(true);
    setIsModalOpen(true);
    getDuplicates([invoice.batchNumber1, invoice.batchNumber2, invoice.batchNumber3]);
  };


  // Function to open the modal for duplicate invoice
  const handleDuplicateInvoiceClick = (invoiceId: string) => {
    const foundInvoice = tableData.find((invoice) => invoice.inv_no === invoiceId);
    if (foundInvoice) {
      setSelectedInvoice(foundInvoice);
      setIsDuplicateModalOpen(true);
    } else {
      setError("Invoice not found.");
    }
  };

  const handleEditClick = () => {
    setIsEditing((prev) => !prev); // Toggle edit mode on "Edit" button click
  };


  // Handler for updating a specific student name
  const handleStudentNameChange = (index: number, value: string) => {
    const updatedNames = [...(formData.studentNames || [])]; // Ensure the array exists
    updatedNames[index] = value; // Update the specific index
    setFormData((prev) => ({
      ...prev,
      studentNames: updatedNames,
    }));
  };
  const handleNumOfClassesChange = (index: number, value: string) => {
    const updatedClasses = [...(formData.numOfClasses || [])];
    updatedClasses[index] = value; // Update the specific index
    setFormData((prev) => ({
      ...prev,
      numOfClasses: updatedClasses,
    }));
  };


  // Handler for adding a new student name field
  const addStudentNameField = () => {
    setFormData((prev) => ({
      ...prev,
      studentNames: [...(prev.studentNames || []), ""],
      numOfClasses: [...(prev.numOfClasses || []), ""], // Add an empty entry
    }));
  };


  // Handler for removing a student name field
  const removeStudentNameField = (index: number) => {
    const updatedNames = (formData.studentNames || []).filter((_, i) => i !== index);
    const updatedClasses = (formData.numOfClasses || []).filter((_, i) => i !== index);
    setFormData((prev) => ({
      ...prev,
      studentNames: updatedNames,
      numOfClasses: updatedClasses,
    }));
  };



  // Generic handler for updating batch information
  const handleBatchInfoChange = (field: keyof InvoiceData, value: string | number) => {
    setFormData((prev) => ({
      ...prev,
      [field]: value, // Dynamically update the batch-related field
    }));
  };



  // Function to close the modal
  const closeModal = () => {
    setIsModalOpen(false);
    // setIsDuplicateModalOpen(false)
    setSelectedInvoice(null);
  };


  // Function to close Duplicate modal

  const closeDuplicateModal = () => {
    setIsDuplicateModalOpen(false)
  }




  useEffect(() => {
    // getDuplicates();
  }, []); // Triggered on component mount

  const getDuplicates = async (batchNumbers: any[]) => {
    setLoading(true);
    setError(null);

    // const batchNumbers = ["401", "402", "403"]; // Example batch numbers, you should replace this with the actual batch numbers

    try {
      // Call getAllDuplicate with the batch numbers to find duplicates
      const response = await getAllDuplicate("invoice", batchNumbers);
      console.log(response, "Hello dup")
      if (response?.duplicates) {
        setDuplicates(response.duplicates); // Assuming the response has a 'duplicates' field
      } else {
        setError("No duplicates found or error in fetching duplicates.");
      }
    } catch (err) {
      setError("Error fetching duplicates.");
    } finally {
      setLoading(false);
    }
  };

  // Open the modal when the "Show Duplicates" button is clicked
  const handleShowDuplicates = () => {
    setIsDuplicateModalOpen(true);
    // getDuplicates(); // Optionally reload duplicates when opening the modal
  };

  // Columns for Ant Design Table
  const columns = [
    {
      title: "Batch Number",
      dataIndex: "batchNumber",
      key: "batchNumber",
    },
    {
      title: "IDs",
      dataIndex: "ids",
      key: "ids",
      render: (ids: string[]) => ids.join(", "),
    },
  ];



  const fetchInvoices = async () => {
    setLoading(true);
    try {
      const response = await axios.get("http://localhost:8000/invoice");
      const sortedData = response.data.data.sort(
        (a: InvoiceData, b: InvoiceData) => b.inv_no.localeCompare(a.inv_no)
      );
      setTableData(response.data.data);
      setFilteredData(response.data.data);
      setLoading(false);
    } catch (error) {
      setError("Failed to fetch invoice data.");
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchInvoices();
  }, []);

  // Handle filtering based on search inputs
  useEffect(() => {
    const filtered = tableData.filter((invoice) => {
      return (
        (!trainerNameFilter ||
          invoice.trainerName
            ?.toLowerCase()
            .includes(trainerNameFilter.toLowerCase())) &&
        (!invNoFilter ||
          invoice.inv_no?.toLowerCase().includes(invNoFilter.toLowerCase()))
      );
    });
    setFilteredData(filtered);
  }, [trainerNameFilter, invNoFilter, tableData]);

  const handleEdit = (invoice: InvoiceData) => {
    console.log("Editing:", invoice);
    // Logic for editing
  };

  const handleDelete = async (inv_no?: string) => {
    const confirmDelete = window.confirm(
      "Are you sure you want to delete this invoice?"
    );
    if (!confirmDelete) return;

    try {
      // Determine which invoice to delete (from table or modal)
      const invoiceNumberToDelete = inv_no || selectedInvoice?.inv_no;

      if (invoiceNumberToDelete) {
        await axios.delete(`http://localhost:8000/invoice/${invoiceNumberToDelete}`);
        alert("Invoice deleted successfully!");
        fetchInvoices(); // Refresh the table
        if (!inv_no) closeModal(); // Close modal only if deletion is triggered from the modal
      } else {
        alert("No invoice selected for deletion.");
      }
    } catch (error) {
      console.error("Error deleting invoice:", error);
      alert("Failed to delete invoice.");
    }
  };

  const handleChange = (
    field: keyof InvoiceData,
    value: string | number | boolean
  ) => {
    setFormData((prev) => ({
      ...prev,
      [field]: value,
    }));
  };

  const handleCalculate = (type: "formData" | "selectedInvoice") => {
    if (type === "formData" && formData) {
      let netPayable = formData.trainerAskedPayment || 0;

      // If TDS is true, apply deduction (e.g., 10% deduction for TDS)
      if (formData.tdsStatus) {
        const tdsDeduction = (formData.trainerAskedPayment || 0) * 0.1; // Example: 10% deduction
        netPayable -= tdsDeduction;
      }

      // Update netPayable in formData
      setFormData((prev) => ({ ...prev, netPayable }));

      // Alert the calculated netPayable
      alert(`Net Payable calculated: ${netPayable}`);
    }

    if (type === "selectedInvoice" && selectedInvoice) {
      let netPayable = selectedInvoice.trainerAskedPayment || 0;

      // If TDS is true, apply deduction (e.g., 10% deduction for TDS)
      if (selectedInvoice.tdsStatus) {
        const tdsDeduction = (selectedInvoice.trainerAskedPayment || 0) * 0.1; // Example: 10% deduction
        netPayable -= tdsDeduction;
      }

      // Update netPayable in selectedInvoice
      setSelectedInvoice((prev) =>
        prev ? { ...prev, netPayable } : null
      );

      // Alert the calculated netPayable
      alert(`Net Payable calculated: ${netPayable}`);
    }
  };

  // Function to style Batch columns dynamically
  const getBatchStyle = (invoice: any, batchKey: string) => ({
    color: duplicates.some(
      (duplicate) =>
        duplicate.ids.includes(invoice.inv_no) &&
        duplicate.batchNumber === `${invoice[batchKey]}`
    )
      ? "red"
      : "inherit",
    fontWeight: duplicates.some(
      (duplicate) =>
        duplicate.ids.includes(invoice.inv_no) &&
        duplicate.batchNumber === `${invoice[batchKey]}`
    )
      ? "bold"
      : "normal",
  });

  const handleStatusChange = async (inv_no: string, newStatus: string) => {
    // Optimistic UI update
    const previousInvoices = [...invoices]; // Save current state for rollback
    const updatedInvoices = invoices.map((invoice) =>
      invoice.inv_no === inv_no ? { ...invoice, status: newStatus } : invoice
    );
    setInvoices(updatedInvoices);
  
    try {
      // Make API call to update status
      const updatedInvoice = await fetchUpdateInvoiceStatus(inv_no, newStatus);
      
  
      // Update the state with the response from the server
      setInvoices((prevInvoices) =>
        prevInvoices.map((invoice) =>
          invoice.inv_no === updatedInvoice.inv_no
            ? { ...invoice, status: updatedInvoice.status }
            : invoice
        )
      );
    } catch (error) {
      console.error("Error updating invoice status:", error);
  
      // Rollback the optimistic UI update
      setInvoices(previousInvoices);
      alert("Failed to update status. Please try again.");
    }
  };
  const getStatusColor = (status: string) => {
    switch (status) {
      case "Approved":
        return "yellow";
      case "Hold":
        return "brown";
      case "Open":
        return "orange";
      case "Receipt Sent":
        return "blue";
      case "Payment Sent":
        return "green";
      default:
        return "white";
    }
  };
  
  

  const handleSubmit = async () => {

    try {
      console.log(formData, "Form Data in submit")
      const result = await axios.post("http://localhost:8000/invoice", formData);
      if (result) {
        alert("Invoice submitted successfully!");
        fetchInvoices();
        setShowForm(false);
        setFormData({
          inv_no: "",
          date: "",
          trainerName: "",
          courseName: "",
          batchNumber1: 0,
          batchNumber2: 0,
          batchNumber3: 0,
          numberOfStudents: 0,
          leaves: 0,
          classes: 0,
          prorata: false,
          numClassesAsMOU: 0,
          costAsPerMOU: 0,
          trainerAskedPayment: 0,
          tds: 0,
          netPayable: 0,
          status: "Open",
          Email: "",
          Contact: "",
          studentNames: [],
          numOfClasses: [],
          paymentPercentage: 0,
          instructor_bank_account: "",
          instructor_contact: "",
          instructor_email: "",
          instructor_pan: "",
          instructor_upi_id: "",
          leavesAmountDeducted: 0,
          mouHours: "",
          tdsStatus: "",
        });
      }

    } catch (error) {
      alert("Failed to submit invoice.");
    }
  };


  interface instructors {
    trainerName: string;
    Email: string;
    Contact: string;
    sno: number;
    key: string;
    label: string;
    value: string;
  }

  const handleChangeCommon = (key: string, value: string | null) => {
    const newFormData = {
      ...formData,
      [key]: value,
    };
    if (key === "email") {
      const instructorDetails = instructorsRef.current.find(
        (instructor) => instructor.Email === value
      );
      newFormData["Contact"] = instructorDetails?.Contact as string;
      newFormData["trainerName"] = instructorDetails?.trainerName as string;
    }
    setFormData(newFormData);
  };

  const handleInstructorSearch: OptionsFilter = ({ options, search }) => {
    if (!search) {
      return instructorsRef.current.map(({ Email, trainerName, Contact }) => ({
        label: Email,
        value: Email,
      }));

    }
    const filteredOptions: ComboboxItem[] = [];

    instructorsRef.current.forEach(({ Email, trainerName, Contact }) => {
      const searchItem = search?.toLowerCase();

      if (
        Email.toLowerCase().includes(searchItem) ||
        trainerName.toLowerCase().includes(searchItem) ||
        Contact.toLowerCase().includes(searchItem)
      ) {
        filteredOptions.push({ label: Email, value: Email });
      }
    });

    return filteredOptions;
  };
  useEffect(() => {
    getInstructors();
  }, []);

  const getInstructors = async () => {
    const response: instructors[] = await getAllInstructorsDetail("instructor");
    instructorsRef.current = response;
    const instructors: ComboboxItem[] = response.map(({ Email }) => ({
      label: Email,
      value: Email,
    }));
    setInstructors(instructors);
  };



  // Fetch the latest invoice number
  const getLatestInvoice = async () => {
    try {
      const response = await fetchLatestInvoice("invoice");
      if (response && response.latestInvoice) {
        const latestInvoiceNo = response.latestInvoice.inv_no || "000";

        // Calculate the next invoice number (increment by 1)
        const nextInvoiceNo = `${String(
          parseInt(latestInvoiceNo.replace("INV", "")) + 1
        ).padStart(3, "0")}`;

        // Update the formData with the calculated invoice number
        setFormData((prev) => ({ ...prev, inv_no: nextInvoiceNo }));
      } else {
        console.warn("No latest invoice found, starting from 01");
        setFormData((prev) => ({ ...prev, inv_no: "001" })); // Default to INV0001 if no data
      }
    } catch (error) {
      console.error("Error fetching the latest invoice:", error);
      setFormData((prev) => ({ ...prev, inv_no: "001" })); // Default to INV0001 on error
    }
  };

  // Show form and fetch the latest invoice when the user clicks the "Create Invoice" button
  useEffect(() => {
    if (showForm) {
      getLatestInvoice();
    }
  }, [showForm]);



  return (
    <div className="invoice-page">
      <h1>Invoice Management</h1>
      <Button className="invoice-btn" onClick={() => setShowForm(!showForm)}>
        {showForm ? "View Invoices" : "Create Invoice"}
      </Button>
      {/* Button to show modal */}
      <button onClick={handleShowDuplicates} disabled={loading}>
        {loading ? "Loading..." : "Show Duplicates"}
      </button>

      <Select
        id="email"
        name="email"
        data={instructors}
        value={formData.Email}
        searchable
        required={true}
        onChange={(val) => handleChangeCommon("Email", val)}
        placeholder="Select Email"
        nothingFoundMessage="Nothing found..."
        filter={handleInstructorSearch}
      />

      {error && <Notification color="red">{error}</Notification>}

      {/* Show Loading State */}
      {loading && <p>Loading duplicates...</p>}


      {showForm ? (
        <div className="invoice-form">
          <h2 style={{ textAlign: "center", marginBottom: "1rem" }}>
            Create Invoice
          </h2>

          <div style={{ display: 'flex', gap: '1rem' }}>
            <TextInput
              label="Invoice No"
              value={formData.inv_no}
              onChange={(e) => handleChange("inv_no", e.target.value)}
              required
              readOnly // Make the invoice number read-only
            />

            {/* Date */}
            <TextInput
              label="Date"
              value={formData.date}
              onChange={(e) => handleChange("date", e.target.value)}
              type="date"
              required
              style={{ flex: 1 }}
            />



            {/* Input field showing dynamic user data */}
            <TextInput
              label="Created By"
              value={`${user?.firstName} ${user?.lastName}`} // Set user name dynamically
              required
              readOnly
              style={{ marginTop: '10px' }}
            />

          </div>

          {/* Trainer Info Section */}
          <div className="box trainer-info">
            <h2>Trainer Info</h2>

            {/* Create a row for the first three inputs */}
            <div style={{ display: 'flex', gap: '1rem' }}>
              <TextInput
                label="Trainer Name"
                value={formData.trainerName || ""}
                onChange={(e) => handleChange("trainerName", e.target.value)}
                style={{ flex: 1 }}
              />
              <TextInput
                label="Trainer Email ID"
                value={formData.instructor_email || ""}
                onChange={(e) => handleChange("instructor_email", e.target.value)}
                style={{ flex: 1 }}
              />
              <TextInput
                label="Trainer Phone No."
                value={formData.instructor_contact || ""}
                onChange={(e) => handleChange("instructor_contact", e.target.value)}
                style={{ flex: 1 }}
              />
            </div>

            {/* Create a row for the next three inputs */}
            <div style={{ display: 'flex', gap: '1rem', marginTop: '1rem' }}>


              <TextInput
                label="Trainer UPI"
                value={formData.instructor_upi_id || ""}
                onChange={(e) => handleChange("instructor_upi_id", e.target.value)}
                style={{ flex: 1 }}
              />
              <TextInput
                label="Trainer PAN"
                value={formData.instructor_pan || ""}
                onChange={(e) => handleChange("instructor_pan", e.target.value)}
                style={{ flex: 1 }}
              />
              <Textarea
                label="Trainer Bank No."
                value={formData.instructor_bank_account || ""}
                onChange={(e) => handleChange("instructor_bank_account", e.target.value)}
                style={{ flex: 1 }}
              />
            </div>

            {/* Checkbox for TDS */}
            <Checkbox
              style={{ margin: '8px' }}
              label="TDS Applicable"
              checked={formData.tdsStatus === "Yes"}  // Check if tdsStatus is "Yes"
              onChange={(e) => handleChange("tdsStatus", e.currentTarget.checked ? "Yes" : "No")}
            />


          </div>


          <Divider my="lg" />

          {/* Student Info Section */}
          <div className="box student-info">
            <h2>Student Info</h2>

            {/* Number of Students */}
            <NumberInput
              label="Number of Students"
              value={formData.numberOfStudents || 0}
              onChange={(value) => handleChange("numberOfStudents", value || 0)}
            />

            {/* Student Details */}
            <Flex direction="row" wrap="wrap" align="center" gap="md">
              {(formData.studentNames || []).map((name, index) => (
                <Group key={index} mt="sm" align="center" grow>
                  {/* Name of Student */}
                  <TextInput
                    label="Name of Student"
                    placeholder={`Student ${index + 1}`}
                    value={name}
                    onChange={(e) => handleStudentNameChange(index, e.target.value)}
                  />

                  {/* Number of Classes */}
                  <TextInput
                    label="No. of Classes"
                    placeholder="Enter number of classes"
                    value={(formData.numOfClasses && formData.numOfClasses[index]) || ""}
                    onChange={(e) => handleNumOfClassesChange(index, e.target.value)}
                  />

                  {/* Remove Student */}
                  <Button
                    color="red"
                    onClick={() => removeStudentNameField(index)}
                  >
                    -
                  </Button>
                </Group>
              ))}
            </Flex>

            {/* Add Student Button */}
            <Button mt="sm" onClick={addStudentNameField}>
              + Add Student
            </Button>
          </div>



          <Divider my="lg" />

          {/* Batch Info Section */}
          <div className="box batch-info">
            <h2>Batch Info</h2>

            {/* Row 1: Batch 1, Batch 2, Batch 3 */}
            <Flex direction="row" wrap="wrap" align="center" gap="md">
              <TextInput
                label="Batch 1"
                value={formData.batchNumber1 || ""}
                onChange={(e) => handleChange("batchNumber1", +e.target.value)}
                style={{ flex: 1 }}
              />
              <TextInput
                label="Batch 2"
                value={formData.batchNumber2 || ""}
                onChange={(e) => handleChange("batchNumber2", +e.target.value)}
                style={{ flex: 1 }}
              />
              <TextInput
                label="Batch 3"
                value={formData.batchNumber3 || ""}
                onChange={(e) => handleChange("batchNumber3", +e.target.value)}
                style={{ flex: 1 }}
              />
            </Flex>

            {/* Row 2: Course Name, No. of Trainer Leaves, No. of Classes */}
            <Flex direction="row" wrap="wrap" align="center" gap="md">
              <TextInput
                label="Course Name"
                value={formData.courseName || ""}
                onChange={(e) => handleChange("courseName", e.target.value)}
                style={{ flex: 1 }}
              />
              <NumberInput
                label="No. of Trainer Leaves"
                value={formData.leaves || 0}
                onChange={(value) => handleChange("leaves", value || 0)}
                style={{ flex: 1 }}
              />
              <NumberInput
                label="No. of Classes"
                value={formData.classes || 0}
                onChange={(value) => handleChange("classes", value || 0)}
                style={{ flex: 1 }}
              />
            </Flex>

            {/* Row 3: Classes As Per MOU, Cost As Per MOU, Trainer Asked for Payment */}
            <Flex direction="row" wrap="wrap" align="center" gap="md">
              <NumberInput
                label="Classes As Per MOU"
                value={formData.numClassesAsMOU || 0}
                onChange={(value) => handleChange("numClassesAsMOU", value || 0)}
                style={{ flex: 1 }}
              />
              <NumberInput
                label="MOU Amount(Per Student)"
                value={formData.costAsPerMOU || 0}
                onChange={(value) => handleChange("costAsPerMOU", value || 0)}
                style={{ flex: 1 }}
              />
              <NumberInput
                label="Trainer Amount (As Per Invoice)"
                value={formData.trainerAskedPayment || 0}
                onChange={(value) => handleChange("trainerAskedPayment", value || 0)}
                style={{ flex: 1 }}
              />
            </Flex>

            {/* Row 4: MOU Hours */}
            <Flex direction="row" wrap="wrap" align="center" gap="md">
              <TextInput
                label="MOU Hours"
                value={formData.mouHours || ""}
                onChange={(e) => handleChange("mouHours", e.target.value)}
                style={{ flex: 1 }}
              />
            </Flex>
          </div>



          <Divider my="lg" />

          <div className="box calculation-info">
            <h2>Calculation</h2>



            {/* Second Row: Custom Calculation, TDS, Payment Percentage */}
            <Flex direction="row" wrap="wrap" align="center" gap="md">
              {/* Custom Calculation Checkbox */}
              <Checkbox
                label="Custom Calculation"
                checked={formData.customCalculation || false}
                onChange={(e) => handleChange("customCalculation", e.currentTarget.checked)}
                style={{ flex: 1 }}
              />

              {/* Conditional TDS Amount Input */}

              <NumberInput
                label="Deduction"
                value={formData.leavesAmountDeducted || 0}
                onChange={(value) => handleChange("leavesAmountDeducted", value || 0)}
                style={{ flex: 1 }}
              />
              {/* Payment Percentage */}
              <NumberInput
                label="Payment Percentage"
                value={formData.paymentPercentage || 0}
                onChange={(value) => handleChange("paymentPercentage", value || 0)}
                style={{ flex: 1 }}
              />
            </Flex>

            {/* First Row: Pro-Rata, Deduction, Net Payable */}
            <Flex direction="row" wrap="wrap" align="center" gap="md">
              <Checkbox
                label="Pro-Rata"
                checked={formData.prorata || false}
                onChange={(e) => handleChange("prorata", e.currentTarget.checked)}
                style={{ flex: 1 }}
              />

              {/* {formData.tds && (
      <NumberInput
        label="TDS Amount"
        value={formData.tdsAmount || 0}
        onChange={(value) => handleChange("tdsAmount", value || 0)}
        style={{ flex: 1 }}
      />
    )} */}
              <NumberInput
                label="Net Payable"
                value={formData.netPayable || 0}
                readOnly
                style={{ flex: 1 }}
              />
            </Flex>


          </div>


          <Divider my="lg" />

          {/* Form Actions */}
          <div
            style={{
              display: "flex",
              // justifyContent: "space-between",
              marginTop: "1.5rem",
            }}
          >
            <Button
              onClick={() => handleCalculate("formData")}
              style={{ marginTop: "1rem" }}
            >
              Calculate
            </Button>

            <Button onClick={handleSubmit}>Submit</Button>
            <Button color="red" onClick={() => setShowForm(false)}>
              Cancel
            </Button>
          </div>

        </div>

      ) : (
        <>
          <div className="search-section">
            <TextInput
              label="Search by Trainer Name"
              placeholder="Enter Trainer Name"
              value={trainerNameFilter}
              onChange={(e) => setTrainerNameFilter(e.target.value)}
            />
            <TextInput
              label="Search by Invoice No"
              placeholder="Enter Invoice No"
              value={invNoFilter}
              onChange={(e) => setInvNoFilter(e.target.value)}
            />
          </div>
          <div className="pagination-controls">
            <Button
              disabled={currentPage === 1}
              onClick={() => setCurrentPage((prev) => Math.max(prev - 1, 1))}
            >
              Previous
            </Button>
            <span>
              Page {currentPage} of {totalPages}
            </span>
            <Button
              disabled={currentPage === totalPages}
              onClick={() => setCurrentPage((prev) => Math.min(prev + 1, totalPages))}
            >
              Next
            </Button>
            <select
              value={rowsPerPage}
              onChange={(e) => {
                setRowsPerPage(Number(e.target.value));
                setCurrentPage(1); // Reset to the first page on rows-per-page change
              }}
            >

              <option value={15}>15 data per page</option>
              <option value={20}>20 data per page</option>
              <option value={40}>40 data per page</option>
              <option value={100}>100 data per page</option>
              <option value={500}>500 data per page</option>
              <option value={1000}>1000 data per page</option>

            </select>
          </div>


          {loading ? (
            <Center>
              <Loader size="xl" />
            </Center>
          ) : error ? (
            <Notification color="red" title="Error">
              {error}
            </Notification>
          ) : (

            <Table
            highlightOnHover
            striped
            style={{  width: "100%", borderCollapse: "collapse" }}
          >
            <thead>
              <tr>
                <th style={{ width: "10%" }}>Invoice No</th>
                <th style={{ width: "15%" }}>Date</th>
                <th style={{ width: "15%" }}>Trainer Name</th>
                <th style={{ width: "10%" }}>Batch1</th>
                <th style={{ width: "10%" }}>Batch2</th>
                <th style={{ width: "10%" }}>Batch3</th>
                <th style={{ width: "10%" }}>Payment</th>
                <th style={{ width: "10%" }}>Net Payable</th>
                <th style={{ width: "10%" }}>Status</th>
                <th style={{ width: "10%" }}>Actions</th>
              </tr>
            </thead>
            <tbody >
              {currentRows.map((invoice) => (
                <tr key={invoice.inv_no}>
                  <td
                   style={{display:"flex", 
                  alignItems:"center",
                    justifyContent:"center",
                  }}>
                    <Button
                      variant="subtle"
                      onClick={() => handleInvoiceClick(invoice)}
                      style={{ cursor: "pointer",margin:"0",padding:"5px ",width:"80px" }}
                    >
                      {invoice.inv_no}
                    </Button>
                  </td>
                  <td >{invoice.date || "N/A"}</td>
                  <td>{invoice.trainerName || "N/A"}</td>
                  <td>{invoice.batchNumber1 || "N/A"}</td>
                  <td>{invoice.batchNumber2 || "N/A"}</td>
                  <td>{invoice.batchNumber3 || "N/A"}</td>
                  <td>{invoice.trainerAskedPayment || "0"}</td>
                  <td>{invoice.netPayable || "0"}</td>
                  <td>
                    <select
                      value={invoice.status}
                      onChange={(e) => handleStatusChange(invoice.inv_no, e.target.value)}
                      style={{
                        padding: "5px",
                        fontSize: "11px",
                        borderRadius: "5px",
                        width: "60px",
                        textAlign: "center",
                        backgroundColor: getStatusColor(invoice.status || "Open"),
                        color: invoice.status === "Hold" ? "black" : "white",
                        fontWeight: "bold",
                        border: "1px solid #ccc",
                        cursor: "pointer",
                      }}
                    >
                      <option value="Approved">Approved</option>
                      <option value="Hold">Hold</option>
                      <option value="Open">Open</option>
                      <option value="Receipt Sent">Receipt Sent</option>
                      <option value="Payment Sent">Payment Sent</option>
                    </select>
                  </td>

                  <td
                 >
                    <Button
                      onClick={() => handleDelete(invoice.inv_no)}
                      color="red"
                      size="xs"
                      style={{
                        backgroundColor:"red",
                        padding: "2px 5px",
                        margin:"0",
                        fontSize: "10px",
                        width: "55px",
                        height: "25px",
                      }}
                    >
                      Delete
                    </Button>
                  </td>
                </tr>
              ))}
            </tbody>
          </Table>
          
          )}


          {/* To Show the Duplicate Entries in Table  */}
          {/* <div style={{ marginTop: "20px" }}>
    <h3>Duplicate Entries</h3>
    {duplicates.length > 0 ? (
      <ul>
        {duplicates.map((duplicate) => (
          <li key={duplicate.batchNumber}>
            <strong>Batch Number:</strong> {duplicate.batchNumber} &rarr;{" "}
            <strong>Invoice Numbers:</strong> {duplicate.ids.join(", ")}
          </li>
        ))}
      </ul>
    ) : (
      <p>No duplicate entries found.</p>
    )}
  </div>  */}
          {/* Modal */}

          {/* Table for displaying duplicates */}
          {/* Modal for duplicates */}
          {isDuplicateModalOpen && (
            <div className="dup-table"
              style={{
                position: "fixed",
                top: "50%",
                left: "50%",
                transform: "translate(-50%, -50%)",
                backgroundColor: "white",
                padding: "20px",
                boxShadow: "0 0 10px rgba(0, 0, 0, 0.5)",
                zIndex: 1000,
              }}
            >
              <h2>Duplicate Invoices</h2>
              <button onClick={closeDuplicateModal} style={{ float: "right", marginBottom: "10px" }}>
                Close
              </button>

              {error && <p style={{ color: "red" }}>{error}</p>}

              <table style={{ width: "100%", borderCollapse: "collapse" }}>
                <thead>
                  <tr className="dup-head" style={{ backgroundColor: "#f2f2f2" }}>
                    <th style={{ border: "1px solid #ddd", padding: "8px" }}>Batch Number</th>
                    <th style={{ border: "1px solid #ddd", padding: "8px" }}>Invoice Numbers</th>
                  </tr>
                </thead>
                <tbody>
                  {duplicates.map((duplicate, index) => (
                    <tr key={index}>
                      <td style={{ border: "1px solid #ddd", padding: "8px" }}>
                        {duplicate.batchNumber}
                      </td>
                      <td style={{ border: "1px solid #ddd", padding: "8px" }}>
                        {duplicate.ids.map((invoiceId, idx) => (
                          <span
                            key={idx}
                            style={{
                              color: "blue",
                              cursor: "pointer",
                              marginRight: "10px",
                              textDecoration: "underline",
                            }}
                            onClick={() => handleDuplicateInvoiceClick(invoiceId)}
                          >
                            {invoiceId}
                          </span>
                        ))}
                      </td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          )}


          <Modal
            opened={isModalOpen}
            onClose={closeModal}
            size="xl"
            scrollAreaComponent={ScrollArea.Autosize}
            title={<span style={{ fontWeight: "bold" }}>Edit Invoice - {selectedInvoice?.inv_no}</span>}
            yOffset="1vh"
            transitionProps={{ transition: 'fade', duration: 600, timingFunction: 'linear' }}
          >
            {selectedInvoice && (
              <div>
                {/* Invoice Info Section */}
                <h2>
                  Invoice Info
                  <a
                    href="#"
                    onClick={handleEditClick} // Edit functionality
                    style={{
                      marginLeft: '10px',
                      fontSize: '14px',
                      color: 'blue',
                      textDecoration: 'none',
                      display: 'inline-flex',
                      alignItems: 'center',
                    }}
                  >
                    <i
                      className={`fas ${isEditing ? 'fa-lock' : 'fa-edit'}`}
                      style={{ marginRight: '5px' }}
                    ></i>
                    {isEditing ? 'Disable Edit' : 'Edit'}
                  </a>
                </h2>
                <Flex direction="row" gap="1rem">
                  <TextInput
                    label="Invoice No"
                    value={selectedInvoice.inv_no}
                    readOnly
                    style={{ flex: 1 }}
                  />
                  <TextInput
                    label="Date"
                    type="date"
                    value={selectedInvoice.date || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, date: e.target.value } : null
                      )
                    }
                    readOnly={!isEditing} // Enable or disable based on editing mode
                    style={{ flex: 1 }}
                  />
                  <TextInput
                    label="Course Name"
                    value={selectedInvoice.courseName || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, courseName: e.target.value } : null
                      )
                    }
                    readOnly={!isEditing} // Enable or disable based on editing mode
                    style={{ flex: 1 }}
                  />
                </Flex>



                <Divider my="lg" />

                {/* Trainer Info Section */}
                <h2>Trainer Info</h2>
                <Flex direction="row" gap="1rem">
                  <TextInput
                    label="Trainer Name"
                    value={selectedInvoice.trainerName || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, trainerName: e.target.value } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                  <TextInput
                    label="Trainer Email"
                    value={selectedInvoice.instructor_email || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, instructor_email: e.target.value } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                  <TextInput
                    label="Trainer Phone"
                    value={selectedInvoice.instructor_contact || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, instructor_contact: e.target.value } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                </Flex>
                <Flex direction="row" gap="1rem" mt="1rem">
                  <TextInput
                    label="Trainer Bank Account"
                    value={selectedInvoice.instructor_bank_account || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, instructor_bank_account: e.target.value } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                  <TextInput
                    label="Trainer UPI ID"
                    value={selectedInvoice.instructor_upi_id || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, instructor_upi_id: e.target.value } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                  <TextInput
                    label="Trainer PAN"
                    value={selectedInvoice.instructor_pan || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, instructor_pan: e.target.value } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                </Flex>


                <Divider my="lg" />

                {/* Student Info Section in Modal */}
                <h2>Student Info</h2>

                {/* Number of Students */}
                <NumberInput
                  label="Number of Students"
                  value={selectedInvoice.numberOfStudents || 0}
                  onChange={(value) =>
                    setSelectedInvoice((prev) =>
                      prev ? { ...prev, numberOfStudents: value as number } : null
                    )
                  }
                  readOnly={!isEditing}
                />

                {/* Student Details */}
                <Flex direction="row" wrap="wrap" align="center" gap="md" mt="1rem">
                  {(selectedInvoice.studentNames || []).map((name, index) => (
                    <Group key={index} mt="sm" align="center" grow>
                      {/* Name of Student */}
                      <TextInput
                        label="Name of Student"
                        placeholder={`Student ${index + 1}`}
                        value={name}
                        onChange={(e) =>
                          setSelectedInvoice((prev) => {
                            if (prev) {
                              const updatedNames = [...(prev.studentNames || [])];
                              updatedNames[index] = e.target.value;
                              return { ...prev, studentNames: updatedNames };
                            }
                            return null;
                          })
                        }
                        readOnly={!isEditing}
                      />

                      {/* No. of Classes */}
                      <TextInput
                        label="No. of Classes"
                        placeholder="Enter number of classes"
                        value={(selectedInvoice.numOfClasses && selectedInvoice.numOfClasses[index]) || ""}
                        onChange={(e) =>
                          setSelectedInvoice((prev) => {
                            if (prev) {
                              const updatedClasses = [...(prev.numOfClasses || [])];
                              updatedClasses[index] = e.target.value;
                              return { ...prev, numOfClasses: updatedClasses };
                            }
                            return null;
                          })
                        }
                        readOnly={!isEditing}
                      />

                      {/* Remove Student */}
                      <Button
                        color="red"
                        onClick={() => {
                          setSelectedInvoice((prev) => {
                            if (prev) {
                              const updatedNames = (prev.studentNames || []).filter(
                                (_, i) => i !== index
                              );
                              const updatedClasses = (prev.numOfClasses || []).filter(
                                (_, i) => i !== index
                              );
                              return {
                                ...prev,
                                studentNames: updatedNames,
                                numOfClasses: updatedClasses,
                              };
                            }
                            return null;
                          });
                        }}
                      >
                        -
                      </Button>
                    </Group>
                  ))}
                </Flex>

                {/* Add Student Button */}
                {isEditing && (
                  <Button
                    mt="sm"
                    onClick={() =>
                      setSelectedInvoice((prev) =>
                        prev
                          ? {
                            ...prev,
                            studentNames: [...(prev.studentNames || []), ""],
                            numOfClasses: [...(prev.numOfClasses || []), ""],
                          }
                          : null
                      )
                    }
                  >
                    + Add Student
                  </Button>
                )}



                <Divider my="lg" />

                {/* Batch Info Section */}
                <h2>Batch Info</h2>
                <Flex direction="row" gap="1rem">
                  <TextInput
                    label="Batch 1"
                    value={selectedInvoice.batchNumber1 || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, batchNumber1: Number(e.target.value) } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                  <TextInput
                    label="Batch 2"
                    value={selectedInvoice.batchNumber2 || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, batchNumber2: Number(e.target.value) } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                  <TextInput
                    label="Batch 3"
                    value={selectedInvoice.batchNumber3 || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, batchNumber3: Number(e.target.value) } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                </Flex>

                <Divider my="lg" />
                {/* Duplicate Info Section */}
                <h2>Duplicate Info</h2>
                {loading ? (
                  <p>Loading duplicate information...</p>
                ) : error ? (
                  <p style={{ color: "red" }}>{error}</p>
                ) : duplicates.length > 0 ? (
                  <div className="dup-section">
                    {/* Table for displaying duplicate invoice details */}
                    <table style={{ width: "100%", borderCollapse: "collapse", marginTop: "1rem" }}>
                      <thead>
                        <tr>
                          <th style={{ border: "1px solid #ccc", padding: "8px", textAlign: "left" }}>
                            Invoice Number
                          </th>
                          <th style={{ border: "1px solid #ccc", padding: "8px", textAlign: "left" }}>
                            Batch Number
                          </th>
                          <th style={{ border: "1px solid #ccc", padding: "8px", textAlign: "left" }}>
                            TDS Amount
                          </th>
                          <th style={{ border: "1px solid #ccc", padding: "8px", textAlign: "left" }}>
                            Net Payable
                          </th>
                        </tr>
                      </thead>
                      <tbody>
                        {duplicates.map((dup) =>
                          dup.ids
                            .filter((id) => id !== selectedInvoice?.inv_no) // Exclude the current invoice
                            .map((invNo, index) => (
                              <tr key={`${dup.batchNumber}-${invNo}-${index}`}>
                                <td style={{ border: "1px solid #ccc", padding: "8px" }}>{invNo}</td>
                                <td style={{ border: "1px solid #ccc", padding: "8px" }}>{dup.batchNumber}</td>
                                <td style={{ border: "1px solid #ccc", padding: "8px" }}>
                                  {/* Replace this placeholder with the actual TDS Amount */}
                                  {/* {dup.tds || "N/A"} */}
                                </td>
                                <td style={{ border: "1px solid #ccc", padding: "8px" }}>
                                  {/* Replace this placeholder with the actual Net Payable */}
                                  {/* {dup.netPayable || "N/A"} */}
                                </td>
                              </tr>
                            ))
                        )}
                      </tbody>
                    </table>
                  </div>
                ) : (
                  <p>No duplicates found for this invoice.</p>
                )}

                <Divider my="lg" />
                {/* <Button onClick={getDuplicates} style={{ marginTop: "10px" }}>
  Refresh Duplicates
</Button> */}



                {/* Payment Info Section */}
                <h2>Payment Info</h2>
                <Flex direction="row" gap="1rem">
                  <TextInput
                    label="Cost As Per MOU"
                    value={selectedInvoice.costAsPerMOU || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, costAsPerMOU: Number(e.target.value) } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                  <TextInput
                    label="Trainer Asked Payment"
                    value={selectedInvoice.trainerAskedPayment || ""}
                    onChange={(e) =>
                      setSelectedInvoice((prev) =>
                        prev ? { ...prev, trainerAskedPayment: Number(e.target.value) } : null
                      )
                    }
                    readOnly={!isEditing}
                    style={{ flex: 1 }}
                  />
                  <TextInput
                    label="Net Payable"
                    value={selectedInvoice.netPayable || ""}
                    readOnly
                    style={{ flex: 1 }}
                  />
                </Flex>

                <Divider my="lg" />

                {/* Calculation Section */}
                <div className="box calculation-info">
                  <h2>Calculation</h2>

                  {/* First Row: Pro-Rata, Deduction, Net Payable */}
                  <Flex direction="row" wrap="wrap" align="center" gap="md">
                    <Checkbox
                      label="Pro-Rata"
                      checked={selectedInvoice.prorata || false}
                      onChange={(e) =>
                        setSelectedInvoice((prev) =>
                          prev ? { ...prev, prorata: e.currentTarget.checked } : null
                        )
                      }
                      readOnly={!isEditing}
                      style={{ flex: 1 }}
                    />

                    <NumberInput
                      label="Net Payable"
                      value={selectedInvoice.netPayable || 0}
                      readOnly
                      style={{ flex: 1 }}
                    />
                  </Flex>

                  {/* Second Row: Custom Calculation, Deduction, Payment Percentage */}
                  <Flex direction="row" wrap="wrap" align="center" gap="md" mt="md">
                    {/* Custom Calculation Checkbox */}
                    <Checkbox
                      label="Custom Calculation"
                      checked={selectedInvoice.customCalculation || false}
                      onChange={(e) =>
                        setSelectedInvoice((prev) =>
                          prev ? { ...prev, customCalculation: e.currentTarget.checked } : null
                        )
                      }
                      readOnly={!isEditing}
                      style={{ flex: 1 }}
                    />

                    {/* Deduction Amount */}
                    <NumberInput
                      label="Deduction"
                      value={selectedInvoice.leavesAmountDeducted || 0}
                      onChange={(value) =>
                        setSelectedInvoice((prev) =>
                          prev
                            ? { ...prev, leavesAmountDeducted: Number(value) || 0 }
                            : null
                        )
                      }
                      readOnly={!isEditing}
                      style={{ flex: 1 }}
                    />

                    {/* Payment Percentage */}
                    <NumberInput
                      label="Payment Percentage"
                      value={selectedInvoice.paymentPercentage || 0}
                      onChange={(value) =>
                        setSelectedInvoice((prev) =>
                          prev
                            ? { ...prev, paymentPercentage: Number(value) || 0 }
                            : null
                        )
                      }
                      readOnly={!isEditing}
                      style={{ flex: 1 }}
                    />

                  </Flex>
                </div>

                {/* Actions */}
                <div style={{ display: 'flex', marginTop: '1.5rem' }}>
                  <Button
                    onClick={async () => {
                      if (selectedInvoice) {
                        try {
                          await axios.put(
                            `http://localhost:8000/invoice/${selectedInvoice.inv_no}`,
                            selectedInvoice
                          );
                          alert("Invoice updated successfully!");
                          fetchInvoices();
                          closeModal();
                        } catch (error) {
                          console.error("Error updating invoice:", error);
                          alert("Failed to update invoice.");
                        }
                      }
                    }}
                  >
                    Update Invoice
                  </Button>
                  {/* Delete Button */}
                  <Button
                    color="red"
                    onClick={() => handleDelete()} // No argument needed, uses selectedInvoice
                    style={{
                      backgroundColor: "red",
                      color: "white",
                    }}
                  >
                    Delete Invoice
                  </Button>


                  <Button color="red" onClick={closeModal} style={{ marginLeft: '1rem' }}>
                    Cancel
                  </Button>
                  {/* <Button
  color="blue"
  onClick={() => setIsDuplicateModalOpen(true)}
>
  View Duplicates
</Button> */}
                  <Button
                    color="green"
                    onClick={() => handleCalculate("selectedInvoice")}
                  >
                    Calculate
                  </Button>


                </div>
              </div>
            )}
          </Modal>

        </>
      )}
    </div>
  );
};

export default InvoicePage;
