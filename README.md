COMPILER BACKEND 


Index
1. Overview
System Purpose
Target Learners
Supported Activities
High-Level Architecture
Compilation Flow (Image → Lex → Parse → Eval → Main)

2. Libraries Used
pyzbar (QR Detection)
OpenCV (Image Processing)
pytesseract (Loop Number OCR)
re (Regex Normalization)

3. Block Types (Lexer Output)
Control Blocks
Condition Blocks
Action Blocks
Colour Blocks
Loop Block
Maze Block
Direction Blocks
Label Blocks


4. Lexical Analysis — lex.py
QR Detection & Sorting
Text Normalization Rules
Block Classification Logic
Lookup Sets
Loop Number Detection (pytesseract)
Lexer Output Format (blocks, loop_count, anchor_x)

5. Parsing — parse.py
Parser Output Structure
Condition Parsing (if / elseif / else)
Colour Pattern Parsing
Loop Expansion
Maze Mode Parsing

6. Evaluation — eval.py
Condition Logic Module


Logic validation
Order validation
Output generation

Pattern / Loop Module


Loop sequence building
Example outputs



7. Maze Navigation Module
Maze Input Format
Finding Start and End
Movement Simulation
Trail Tracking
Collision & Failure Detection
Structured Output Format




8. main.py — Module Routing
Module Detection Rules
Maze Mode Trigger
Condition Mode Trigger
Pattern/Loop Mode Trigger

9. System Diagrams
Complete Compiler Pipeline
Maze Module Execution Flow

10. Extending the Compiler
Adding Lexical Tokens
Adding Parsing Logic
Adding New Evaluators
Updating main.py Router

11. Examples (End-to-End)
Condition Module Example
Loop Module Example
Maze Module Example
Block Pattern Examples
Maze
Loop
If-Else



1. Overview
The CT4PWD Block Compiler is a vision-based block programming system.
 Users arrange QR-based physical blocks (if, loop, colour, direction, etc.),
 The system captures them using a camera and compiles them into executable logic.
System goals:
Teach programming concepts visually to students on the autism spectrum.


Support multiple “activities”:


Conditional logic


Pattern creation (colour sequences)


Loops


Maze navigation


High-level flow:
Image → lex.py (QR detection) 
      → parse.py (build logical sequence)
      → eval.py (execute activity logic)
      → main.py (route output)


2. Libraries Used
🔹 pyzbar
Used for QR code detection.
from pyzbar.pyzbar import decode

🔹 OpenCV (cv2)
Used for reading images, cropping regions, preprocessing text for loops.

🔹 pytesseract
Used only in one place: reading loop numbers from cropped region near “loop” QR.

🔹 re
Used for normalization and cleaning QR text.
Everything else is pure Python (lists, dicts, loops).

3. Block Types (What the Lexer Produces)
type values that lex.py emits:
Block Type
Meaning
Example
"control"
if / elseif / else
"if"
"condition"
raining, sunny, red, etc.
"raining"
"action"
umbrella, stop, go
"stop"
"colour"
red, green, blue
"blue"
"loop"
loop count
loop x3
"maze"
maze mode triggered
"maze"
"direction"
up/down/left/right
"right"
"label"
fallback text
anything unrecognized


4. How lex.py Works (Lexical Analysis)
lex.py converts QR codes into structured blocks.
Steps:
Read image and detect QRs:
qr_codes = decode(image)
qr_codes.sort(key=lambda q: q.rect.left)

Sorting ensures left-to-right execution order.

Normalize the QR text
"Else If" → "elseif"
"  GREEN!! " → "green"

Classify block
Depending on QR content, one of:
{"type": "control", "value": "if"}
{"type": "condition", "value": "red"}
{"type": "loop", "value": 3}
{"type": "direction", "value": "up"}
{"type": "maze", "value": "maze"}

The classification logic uses lookup sets:
CONTROL_SET
CONDITION_SET
ACTION_SET
COLOUR_WORDS
DIRECTION_SET

Loop block does extra work:
It crops a region to the right of the QR → reads number → stores loop count.
Result returned to parser:
blocks: list of block dicts
loop_count: extracted from a “loop” block
anchor_x: rightmost coordinate (not used further)


5. How parse.py Works (Parsing / Structure Builder)
The parser’s job is to assemble a logical program out of the block list.
Key outputs:
{
  "colours": [],
  "loop_count": 1,
  "conditions": [],
  "sequence": [...]
}


Condition Parsing (if / elseif / else)
The parser uses helper functions:
_parse_if()


_parse_elseif()


_parse_else()


Each consumes 2–3 blocks and returns:
Example:
Blocks:
if raining stop

Parsed entry:  {"if": "raining", "action": "stop"}

Sequence tokens:
["if", "raining", "stop"]


Colour / Pattern Parsing
A simple append:
if t == "colour":
    colours.append(v)
    sequence.append(v)


Loop Handling
If a loop block was detected:
sequence × loop_count

So:
     ["red", "green"] × 3

→ ["red", "green", "red", "green", "red", "green"]


Maze Mode Parsing (Custom Extension)
Maze and direction blocks do NOT require transformation.
 They are appended as-is:
if t in {"maze", "direction"}:
    sequence.append(b)


6. How eval.py Works (Evaluation Logic)
The evaluator chooses logic based on parsed content:

Condition Logic Module
Steps:
Validate logic:


red → stop
green → go
If mismatch:
incorrect logic: red-go
correct: red-stop


      2. Validate order:

 	if → elseif → else
      3. Output sequence:
	if red stop elseif green go else umbrella


Pattern / Loop Module
If no condition blocks are present:
Output: repeated colour sequence


Example:
colours = ["red", "blue"]
loop = 3
Output → "red blue red blue red blue"


7. Maze Navigation Module
Triggered by:
{"type":"maze","value":"maze"}

Flow:
main.py detects maze mode.


It asks the user to input maze text (S, E, 0, 1) (‘S’=starting cell, ‘E’=ending cell, ‘1’=obstacle, ‘0’=clear path)


simulate_maze() does:


Parse maze
Input:
S 0 1
0 0 0
1 0 E

→ converted to 2D grid.

Find start (S) and end (E)

Simulate movement
Commands: right, down, down, right

Moves the pointer step-by-step.
Tracks:
trail: list of coordinates visited


directions_traversed: ["right", "down", ...]


point_of_failure: coordinate if collision


direction_of_collision: arrow


success or failure message



Failure Conditions
Out of bounds


Hit a wall (1)


Did not reach ‘E’ within commands


Errors returned in structured format:
{
  "result": "❌ Hit a wall at step 3",
  "trail": [(0,0),(0,1),(1,1)],
  "directions_traversed": ["right","down"],
  "point_of_failure": [1,2],
  "direction_of_collision": "right"
}


8. main.py (The Router)
main.py determines which module to run.
8.1 Detect module
if first block is maze:
    		call simulate_maze()

elif conditions present:
    		call condition evaluator
else:
    		call pattern/loop evaluator


9. System Diagrams

9.1 Full Pipeline



9.2 Maze Module Flow
sequence starts with "maze"
        ↓
main.py → ask user for maze
        ↓
simulate_maze()
        ↓
trail + failure + commands
        ↓
main.py prints the result


10. Extending the Compiler
To add new modules:
Step 1 — Add new recognition to lex.py
Add new keyword to sets:
NEW_SET = {...}

Step 2 — Add parsing rules
Either:
append raw blocks, or


write a specialized parser


Step 3 — Add evaluator
One function inside eval.py.
Step 4 — Add router logic in main.py
Detect the module, call evaluator.
This modular structure makes your compiler very extensible.

11. Examples
11.1 Condition Example
if → red → stop → else → umbrella

Output:
if red stop else umbrella


11.2 Loop Example
QR:
Loop 3 red blue

Output:
red blue red blue red blue


11.3 Maze Example
Blocks:
maze right down right right down

Maze:
S 0 0 1
1 0 0 0
1 1 0 E

Output:
{
 "result": "Level cleared",
 "trail": [(0,0),(0,1),(1,1),(1,2),(1,3),(2,3)],
 "directions_traversed": ["right","down","right"],
 "point_of_failure": null
}




11.4 Block pattern examples:

Maze module:


Loop module:


If-else module:



Developed by: Aditya Gupta (2022A7PS0090G)
