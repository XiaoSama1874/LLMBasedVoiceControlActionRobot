# VoiceControlRobot

Voice-controlled robot system for 639 Introduction to Robotics final project.

## Project Overview

This project implements a voice-controlled robot system that:
1. Listens to voice commands continuously
2. Uses LLM (GPT-5) to parse natural language commands into structured execution plans
3. Executes plans using vision recognition and robotic arm control
4. Maintains context between commands for relative movements

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Voice Input                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Voice Module                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Speech       │  │ Command      │  │ Voice        │          │
│  │ Recognizer   │→ │ Detector     │→ │ Listener     │          │
│  │ (Google STT)│  │ (Keywords +  │  │ (Orchestrator)│          │
│  │              │  │  Silence)   │  │              │          │
│  └──────────────┘  └──────────────┘  └──────┬───────┘          │
└───────────────────────────────────────────────┼─────────────────┘
                                                 │ Detected Command
                                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      LLM Module                                 │
│  ┌──────────────┐  ┌──────────────┐                           │
│  │ LLM Planner  │→ │ Plan Parser   │                           │
│  │ (GPT-5)      │  │ (JSON Valid.) │                           │
│  └──────────────┘  └──────┬───────┘                           │
└────────────────────────────┼────────────────────────────────────┘
                             │ Execution Plan (JSON)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Executor Module                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Executor     │→ │ Robot        │  │ Context      │         │
│  │ (Plan Runner)│  │ Functions    │  │ Manager      │         │
│  │              │  │ (move/grasp/ │  │ (State)      │         │
│  │              │  │  see/move_   │  │              │         │
│  │              │  │  home)       │  │              │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Robot Hardware │
                    │ (Arm + Vision) │
                    └────────────────┘
```

## Supported Capabilities

### 1. Voice Recognition
- **Continuous Listening**: Monitors microphone input continuously
- **Speech-to-Text**: Uses Google Speech Recognition API (English)
- **Command Detection**: 
  - Start keyword: "hi robot" (configurable)
  - Stop keyword: "command end" (configurable)
  - **Silence Detection**: Automatically ends command after 2 seconds of silence (configurable)
- **Multi-segment Commands**: Supports commands split across multiple speech segments

### 2. Natural Language Understanding
- **LLM Integration**: Uses OpenAI GPT-5 to understand natural language commands
- **Plan Generation**: Converts commands into structured JSON execution plans
- **Smart Planning**: Automatically adds necessary steps (e.g., move_home before see())

### 3. Relative Movement
- **Direction-based Movement**: Supports relative movement commands
  - Directions: `right`, `left`, `forward`, `backward`, `back`, `up`, `down`
  - Mapping: right→x+, left→x-, forward→y+, backward/back→y-, up→z+, down→z-
- **Default Distance**: 20cm (configurable in `config.py`)
- **Context Preservation**: Maintains robot position between commands
  - Example: "move forward" followed by "move forward" will move 20cm + 20cm from initial position
- **Usage Examples**:
  - "move the robot arm to the right"
  - "grab the red cube and move it forward"
  - "move forward 30cm" (custom distance)

### 4. Vision Recognition
- **Object Detection**: Identifies targets using vision system
- **Automatic Home Positioning**: Automatically moves to home position before vision recognition
- **Coordinate Extraction**: Returns target coordinates for subsequent movements
- **Supported Targets**: red_cube, blue_cube, green_cube (configurable in `config.py`)

### 5. Context Management
- **Position Tracking**: Maintains robot position across multiple commands
- **State Persistence**: Preserves grasp state, vision results, and position between executions
- **Data Flow**: Automatically passes vision coordinates to subsequent move actions

### 6. Error Handling
- **Robust Execution**: Handles errors gracefully with detailed feedback
- **Retry Mechanism**: LLM plan generation includes retry logic
- **Validation**: Plans are validated before execution

## Project Structure

```
VoiceControlRobot/
├── config.py                    # Centralized configuration file
│                                # - Voice settings (keywords, language, timeouts)
│                                # - LLM settings (model, temperature, tokens)
│                                # - Robot settings (home position, relative move distance)
│                                # - Direction mappings
│
├── main_voice.py               # Main entry point for voice-controlled system
│
├── requirements.txt            # Python dependencies
│
├── voice_module/               # Voice recognition and command detection
│   ├── __init__.py
│   ├── speech_recognizer.py    # Google Speech Recognition wrapper
│   ├── command_detector.py     # Command boundary detection (keywords + silence)
│   └── voice_listener.py      # Main listening orchestrator
│
├── llm_module/                 # LLM-based plan generation
│   ├── __init__.py
│   ├── llm_planner.py         # GPT-5 integration for plan generation
│   └── plan_parser.py         # JSON plan parsing and validation
│
├── executor_module/            # Plan execution and robot control
│   ├── __init__.py
│   ├── executor.py            # Main executor (runs plans, manages context)
│   └── robot_functions.py     # Atomic robot functions (move, grasp, see, move_home)
│
└── test/                       # Unit tests
    ├── __init__.py
    ├── test_voice_module.py   # Voice module tests
    ├── test_llm_module.py     # LLM module tests
    ├── test_executor_module.py # Executor module tests
    ├── test_plan_parser.py    # Plan parser tests
    ├── test_full_pipeline.py  # End-to-end pipeline tests
    └── llm_plans/             # Generated execution plans from tests
        └── *.txt              # Saved LLM-generated plans
```

### Directory Descriptions

#### `config.py`
Central configuration file containing all hyperparameters:
- Voice module settings (keywords, language, timeouts, energy thresholds)
- LLM module settings (model, temperature, max tokens, retries)
- Robot settings (home position, default positions, execution delays)
- Relative movement settings (default distance, direction mappings)
- Valid robot actions

#### `voice_module/`
Handles all voice-related functionality:
- **speech_recognizer.py**: Wraps Google Speech Recognition API
- **command_detector.py**: Detects command boundaries using start/stop keywords and silence detection
- **voice_listener.py**: Orchestrates continuous listening, recognition, and command detection

#### `llm_module/`
Converts natural language commands into execution plans:
- **llm_planner.py**: Interfaces with OpenAI GPT-5 to generate execution plans
- **plan_parser.py**: Parses and validates JSON execution plans from LLM

#### `executor_module/`
Executes plans and controls the robot:
- **executor.py**: Main execution engine that:
  - Runs tasks in sequence
  - Manages execution context (position, grasp state, vision results)
  - Handles relative movement calculations
  - Ensures home position before vision recognition
  - Preserves context between command executions
- **robot_functions.py**: Placeholder implementations for robot atomic functions:
  - `move_home()`: Move to home position
  - `move(x, y, z)`: Move to absolute coordinates
  - `grasp(grasp)`: Grasp (True) or release (False)
  - `see(target)`: Vision recognition

#### `test/`
Comprehensive unit tests for all modules:
- Individual module tests
- Integration tests
- Full pipeline tests
- Generated LLM plans saved in `llm_plans/` subdirectory

## Installation

1. Install dependencies:
```bash
pip install -r requirements.txt
```

Note: On macOS, you may need to install PortAudio first:
```bash
brew install portaudio
```

2. Set up environment variables:
```bash
cp .env.example .env
# Edit .env and add your OpenAI API key
```

## Usage

### Running the Voice-Controlled System

```bash
python main_voice.py
```

**Instructions:**
1. Say "hi robot" to start a command
2. Say your command (e.g., "pick up the red cube and move it to the right")
3. Say "command end" to end the command, OR wait 2 seconds of silence
4. The system will generate an execution plan and execute it
5. Press Ctrl+C to exit

### Example Commands

- **Simple movement**: "move the robot arm forward"
- **Relative movement**: "move the robot arm to the right 30cm"
- **Object manipulation**: "pick up the red cube"
- **Complex task**: "grab the blue cube and move it forward, then release it"

## Testing

### Running Tests

Run all unit tests:
```bash
python -m unittest discover test
```

Run with verbose output:
```bash
python -m unittest discover test -v
```

Run specific test file:
```bash
python -m unittest test.test_llm_module
python -m unittest test.test_executor_module
python -m unittest test.test_voice_module
python -m unittest test.test_plan_parser
python -m unittest test.test_full_pipeline
```

### Test Files

1. **test/test_llm_module.py** - Tests for LLM plan generation
2. **test/test_executor_module.py** - Tests for plan execution
3. **test/test_voice_module.py** - Tests for voice module
4. **test/test_plan_parser.py** - Tests for plan parsing
5. **test/test_full_pipeline.py** - End-to-end pipeline tests

### Note on Voice Tests

Tests that require microphone input are marked with `@unittest.skip` by default. To run them, remove the decorator.

## Configuration

All configuration is centralized in `config.py`. Key settings:

### Voice Module
- `VOICE_START_KEYWORD`: Command start keyword (default: "hi robot")
- `VOICE_STOP_KEYWORD`: Command stop keyword (default: "command end")
- `VOICE_COMMAND_END_SILENCE_DURATION`: Silence timeout in seconds (default: 2.0)
- `VOICE_LANGUAGE`: Speech recognition language (default: "en-US")

### LLM Module
- `LLM_MODEL`: OpenAI model to use (default: "gpt-5.1")
- `LLM_TEMPERATURE`: Sampling temperature (default: 0.7)
- `LLM_MAX_TOKENS`: Maximum response tokens (default: 2000)

### Robot Settings
- `ROBOT_HOME_POSITION`: Home position coordinates (default: {0, 0, 0})
- `ROBOT_RELATIVE_MOVE_DEFAULT_DISTANCE`: Default relative move distance in cm (default: 20.0)
- `ROBOT_DIRECTION_MAPPING`: Direction to axis mapping for relative movement

## Execution Plan Format

Execution plans are JSON structures with the following format:

```json
{
  "tasks": [
    {
      "task_id": 0,
      "action": "move_home",
      "description": "Move to home position for vision recognition"
    },
    {
      "task_id": 1,
      "action": "see",
      "description": "Identify the red cube using vision",
      "parameters": {"target": "red_cube"}
    },
    {
      "task_id": 2,
      "action": "move",
      "description": "Move arm to red cube location",
      "parameters": {"x": null, "y": null, "z": null}
    },
    {
      "task_id": 3,
      "action": "grasp",
      "description": "Grasp the red cube",
      "parameters": {"grasp": true}
    },
    {
      "task_id": 4,
      "action": "move",
      "description": "Move robot arm to the right",
      "parameters": {"relative": true, "direction": "right", "distance": 20}
    }
  ]
}
```

### Supported Actions

1. **move_home** - Move robot arm to home position (no parameters)
2. **move** - Move robot arm:
   - Absolute: `{"x": value, "y": value, "z": value}`
   - Relative: `{"relative": true, "direction": "right|left|forward|backward|back|up|down", "distance": value}`
3. **grasp** - Grasp or release: `{"grasp": true|false}`
4. **see** - Vision recognition: `{"target": "target_name"}`

## Key Features

### Automatic Home Positioning
- Vision recognition (`see()`) automatically moves robot to home position if not already there
- LLM is instructed to add `move_home()` before `see()` actions

### Context Preservation
- Robot position is maintained between command executions
- Relative movements are calculated based on the last known position
- Vision results and grasp state are preserved in execution context

### Smart Plan Generation
- LLM automatically adds necessary steps (e.g., move_home before see)
- Handles relative movement commands intelligently
- Validates plans before execution

## Development Notes

- Robot functions (`move`, `grasp`, `see`, `move_home`) are placeholder implementations
- Actual hardware integration will be provided by team members
- All configuration is centralized in `config.py` for easy modification
- Comprehensive unit tests ensure system reliability

## License

See LICENSE file for details.
