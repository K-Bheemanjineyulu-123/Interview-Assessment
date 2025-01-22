# Interview-Assessment


# Backend: FastAPI with LangGraph and Open-Source LLM

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate
from langchain.llms import HuggingFaceHub
import mysql.connector

app = FastAPI()

# Database connection
DB_CONFIG = {
    "host": "localhost",
    "user": "root",
    "password": "password",
    "database": "chatbot_db"
}

class QueryRequest(BaseModel):
    query: str

# Initialize LLM (e.g., HuggingFace GPT-2)
llm = HuggingFaceHub(repo_id="gpt2", model_kwargs={"temperature": 0.5})

prompt = PromptTemplate(
    input_variables=["query", "data"],
    template="User query: {query}\nDatabase data: {data}\nAnswer the query based on the data."
)

@app.post("/query")
def handle_query(request: QueryRequest):
    query = request.query

    try:
        # Connect to database
        conn = mysql.connector.connect(**DB_CONFIG)
        cursor = conn.cursor(dictionary=True)

        # Handle specific queries
        if "products under brand" in query.lower():
            brand = query.split("brand")[-1].strip()
            cursor.execute("SELECT * FROM Products WHERE brand = %s", (brand,))
        elif "suppliers provide" in query.lower():
            product = query.split("provide")[-1].strip()
            cursor.execute("SELECT * FROM Suppliers WHERE FIND_IN_SET(%s, product_categories)", (product,))
        elif "details of product" in query.lower():
            product = query.split("product")[-1].strip()
            cursor.execute("SELECT * FROM Products WHERE name = %s", (product,))
        else:
            return {"answer": "I couldn't understand your query. Please rephrase."}

        # Fetch data
        data = cursor.fetchall()
        if not data:
            return {"answer": "No matching data found."}

        # Summarize data with LLM
        chain = LLMChain(llm=llm, prompt=prompt)
        summary = chain.run(query=query, data=str(data))
        return {"answer": summary}

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

    finally:
        cursor.close()
        conn.close()

# Frontend: React Component
// ChatWindow.jsx
import React, { useState } from "react";
import { Box, TextField, Button, Typography, Paper } from "@mui/material";
import axios from "axios";

const ChatWindow = () => {
    const [query, setQuery] = useState("");
    const [responses, setResponses] = useState([]);

    const handleQuerySubmit = async () => {
        if (!query) return;

        try {
            const response = await axios.post("http://localhost:8000/query", { query });
            setResponses((prev) => [...prev, { user: query, bot: response.data.answer }]);
            setQuery("");
        } catch (error) {
            console.error("Error fetching data:", error);
            setResponses((prev) => [...prev, { user: query, bot: "Error retrieving data." }]);
        }
    };

    return (
        <Box p={2}>
            <Typography variant="h4" gutterBottom>
                AI-Powered Chatbot
            </Typography>
            <Paper elevation={3} style={{ maxHeight: "400px", overflowY: "auto", padding: "10px" }}>
                {responses.map((res, index) => (
                    <Box key={index} mb={2}>
                        <Typography variant="body1"><strong>You:</strong> {res.user}</Typography>
                        <Typography variant="body2" style={{ color: "#3f51b5" }}><strong>Bot:</strong> {res.bot}</Typography>
                    </Box>
                ))}
            </Paper>
            <TextField
                fullWidth
                label="Ask a question"
                value={query}
                onChange={(e) => setQuery(e.target.value)}
                variant="outlined"
                margin="normal"
            />
            <Button variant="contained" color="primary" onClick={handleQuerySubmit}>
                Submit
            </Button>
        </Box>
    );
};

export default ChatWindow;

// App.js
import React from "react";
import ChatWindow from "./ChatWindow";

function App() {
    return <ChatWindow />;
}

export default App;

# Database Schema (MySQL)
CREATE DATABASE chatbot_db;

USE chatbot_db;

CREATE TABLE Products (
    ID INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    brand VARCHAR(50),
    price DECIMAL(10, 2),
    category VARCHAR(50),
    description TEXT,
    supplier_id INT
);

CREATE TABLE Suppliers (
    ID INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    contact_info VARCHAR(150),
    product_categories TEXT
);

INSERT INTO Products (name, brand, price, category, description, supplier_id)
VALUES
('Laptop X', 'BrandX', 1000.00, 'Electronics', 'High-performance laptop', 1);

INSERT INTO Suppliers (name, contact_info, product_categories)
VALUES
('Supplier A', 'contact@supplierA.com', 'Electronics, Laptops');
