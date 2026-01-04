
# 📜 Submission Rules

To contribute, create a folder under:

01-fundamentals/projects/[your-github-username]/levels

Include:
- Your code
- A short **REASONING.md** explaining your architectural choices

---

## 🟢 Level 1: Simple (Testing Syntax)

### ATM Simulator
A CLI tool that:
- Asks for a PIN  
- Allows **deposits** and **withdrawals** (stored in a variable)  
- Prevents withdrawing more than the available balance  

### Password Generator
A script that:
- Takes user input for password length  
- Accepts a "strength" option (include symbols/numbers)  
- Outputs a secure random string  

### Unit Converter
A tool that converts between:
- Celsius ↔ Fahrenheit  
- Kilograms ↔ Pounds  
- Meters ↔ Feet  

Each conversion must be implemented using separate functions.

---

## 🟡 Level 2: Intermediate (Testing Logic & Data)

### Markdown-to-HTML Converter
A script that:
- Reads a simple `.md` file  
- Converts:
  - `# Heading` → `<h1>Heading</h1>`
  - `**bold**` → `<b>bold</b>`  
- Saves the output to an `.html` file  

### Expense Tracker
A program where users can:
- Add expenses with categories (Food, Rent, Fun)  
- Save data to a `.json` file  
- Calculate total spending per category  

### Contact Book (OOP)
Use **Classes** to create a `Contact` object.

The program must:
- Add contacts  
- Delete contacts  
- Search contacts  
- Update contacts  
- Prevent duplicate phone numbers  

---

## 🔴 Level 3: Hard (Testing Systems & Architecture)

### Task Management System (CLI)

**The Challenge:**  
Build a Todo app that supports:
- Priority Levels (High, Medium, Low)  
- Due Dates  

**Hard Twist:**  
- Allow users to **sort tasks** by date or priority  
- Implement **file persistence** so tasks remain after restarting the program  

---

### The "Dungeon Crawler" Text Game

**The Challenge:**  
Create a game where a player moves through rooms:
- North
- South
- East
- West  

**Hard Twist:**  
- Use a **Dictionary** to map rooms  
- Create:
  - `Enemy` class  
  - `Player` class  
- Implement turn-based combat  
- Calculate stats (Health, Attack) using the `random` module  

---

### Student Grading System (Analytics)

**The Challenge:**  
- Read a CSV file containing 100+ students  
- Each student has scores in 5 subjects  

**Hard Twist:**  
- Calculate GPA  
- Find the top 3 students  
- Identify students failing more than 2 subjects  
- Automatically export a **Report Card** text file for every student  

```
projects/
└── [your-github-username]/
    └── levels/
        ├── level-1-simple/
        │   ├── atm-simulator/
        │   │   ├── main.py
        │   │   └── REASONING.md
        │   │
        │   ├── password-generator/
        │   │   ├── main.py
        │   │   └── REASONING.md
        │   │
        │   └── unit-converter/
        │       ├── main.py
        │       └── REASONING.md
        │
        ├── level-2-intermediate/
        │   ├── markdown-to-html/
        │   │   ├── input.md
        │   │   ├── output.html
        │   │   ├── converter.py
        │   │   └── REASONING.md
        │   │
        │   ├── expense-tracker/
        │   │   ├── expenses.json
        │   │   ├── tracker.py
        │   │   └── REASONING.md
        │   │
        │   └── contact-book/
        │       ├── contact.py
        │       ├── app.py
        │       └── REASONING.md
        │
        └── level-3-hard/
            ├── task-manager-cli/
            │   ├── tasks.json
            │   ├── main.py
            │   └── REASONING.md
            │
            ├── dungeon-crawler/
            │   ├── player.py
            │   ├── enemy.py
            │   ├── map.py
            │   ├── game.py
            │   └── REASONING.md
            │
            └── student-grading-system/
                ├── students.csv
                ├── grading.py
                ├── report_cards/
                │   ├── student_001.txt
                │   ├── student_002.txt
                │   └── ...
                └── REASONING.md
```