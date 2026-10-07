✨ What the Project Does

The AI Real-Time Gym Coach turns a normal webcam into an interactive fitness assistant.

It observes the user's movements through the camera and extracts body-pose information such as:

Repetition count

Joint angles

Movement stage

Exercise depth

Body alignment

Balance

Swing detection

Back arch

Exercise-specific form conditions

These measurements are displayed live in the Streamlit application.

The project also has an AI coaching layer. Instead of asking the LLM to perform pose detection, the computer-vision system first identifies what is happening. The LLM then converts important workout events or form issues into short, natural coaching instructions.

🚀 Main Highlights

Real-Time Pose Tracking

Camera frames are received through streamlit-webrtc and processed continuously using MediaPipe Pose Landmarker.

Exercise Recognition & Analysis

Each supported exercise has its own detector containing movement and form rules.

Smart Rep Counting

Repetitions are based on movement-state transitions rather than simply checking individual frames, helping avoid duplicate counts.

Form Monitoring

The system checks exercise-specific technique indicators such as knee angles, elbow position, hip alignment, torso movement, depth, balance, and back arch.

AI-Powered Coaching

Workout events and detected form problems can be sent to a Groq-hosted LLM to create concise coaching cues.

Voice Feedback

The generated coaching message is converted into speech using Google Text-to-Speech (gTTS) and played through the Streamlit interface.

Workout Persistence

User information, workouts, and workout history are maintained using SQLite.

🏃 Supported Exercises

Exercise

Rep Measurement

Main Form Checks

Squats

Knee angle + movement stage

Knee angle, back angle, depth

Push-ups

Elbow angle + movement stage

Body alignment, elbow angle, hip position

Biceps Curls

Elbow angle + movement stage

Elbow drift, shoulder stability, swing

Shoulder Press

Elbow angle + movement stage

Arm extension, elbow angle, back arch

Lunges

Front knee angle + movement stage

Knee angle, torso angle, balance

🔄 Application Workflow

Browser Camera
      ↓
Streamlit WebRTC
      ↓
MediaPipe Pose Landmarker
      ↓
Body Pose Landmarks
      ↓
Exercise Detector
      ↓
Rep Counting + Form Analysis
      ↓
Workout Event
      ↓
Groq LLM
      ↓
Coaching Message
      ↓
gTTS
      ↓
Voice Feedback
      ↓
Workout History

The important design principle is that computer vision handles movement analysis, while the LLM handles natural-language coaching.

🧠 Computer Vision Pipeline

For every incoming camera frame, the application performs approximately these steps:

Receives the camera frame.

Converts it into an OpenCV-compatible format.

Horizontally flips the frame.

Runs MediaPipe Pose Landmarker.

Extracts body landmarks.

Selects the active exercise detector.

Calculates exercise-specific metrics.

Draws the skeleton and feedback information.

Updates the latest workout metrics.

Sends the processed frame back to the browser.

The pose model is located at:

ml_models/pose_landmarker_full.task

Current pose configuration:

Running Mode: VIDEO
Minimum Detection Confidence: 0.7
Minimum Pose Presence Confidence: 0.7
Minimum Tracking Confidence: 0.7
Segmentation Masks: Disabled

Most exercise detectors use a landmark visibility threshold close to:

MIN_VISIBILITY = 0.7

💪 Exercise Detection Logic

Each workout movement has an independent detector.

detectors/
├── squat.py
├── pushup.py
├── biceps_curl.py
├── shoulder_press.py
└── lunges.py

Squats

Tracks:

Left and right knee angles

Selected knee angle

Back angle

Repetitions

Squat depth

Typical movement:

Knee angle < 100°
       ↓
     DOWN
       ↓
Knee angle >= 160°
       ↓
      UP
       ↓
    REP + 1

Depth can be reported as GOOD DEPTH, TOO HIGH, STANDING, or N/A.

Push-ups

Tracks:

Elbow angle

Body angle

Hip deviation

Repetitions

Body alignment

Hip position

Typical movement:

Elbow angle < 90°
       ↓
     DOWN
       ↓
Elbow angle > 160°
       ↓
      UP
       ↓
    REP + 1

The system can identify conditions such as straight alignment, slight bending, poor form, sagging, and a piked-up position.

Biceps Curls

The detector chooses the arm with better elbow visibility and monitors:

Elbow angle

Elbow drift

Torso movement

Repetitions

Movement states are based on curl and extension angles.

Form feedback can include:

STABLE / ELBOW DRIFTING
NO SWING / SWINGING

Shoulder Press

Measures:

Elbow angle

Arm extension

Back angle

Repetitions

It can identify:

FULL EXTENSION
NEARLY EXTENDED
PRESSING
START POSITION

Back posture can also be classified as neutral, slightly arched, or excessively arched.

Lunges

The detector evaluates both knees and uses the smaller knee angle as the front-knee measurement.

Tracked values include:

Front knee angle

Torso angle

Balance

Repetitions

Balance feedback can be:

BALANCED
OFF BALANCE

🤖 AI Coaching System

The AI coaching system is intentionally separated from pose analysis.

Vision System

Answers:

What is the user doing?

AI Coaching System

Answers:

How should the feedback be communicated?

Workout events may include:

workout_started
set_completed
workout_completed
no_pose_detected
ongoing_form_check

Examples of detected issues include:

Squat depth is too high

Excessive forward leaning

Push-up hip sagging

Push-up hips raised too much

Biceps curl swinging

Elbow drifting during curls

Excessive shoulder-press back arch

Losing balance during lunges

🗣️ LLM + Voice Pipeline

A compact event is sent to the Groq LLM when coaching feedback is required.

Example:

Event: ongoing_form_check
Form Issue: The user's hips are sagging during the push-up.

The model receives the system instructions, recent conversation context, current event, and detected form issue.

The project uses:

llama-3.3-70b-versatile

through Groq.

The coaching prompt is designed for short spoken feedback, approximately 10–15 words, with a natural, energetic, exercise-specific, and safety-conscious tone.

The generated text then follows:

LLM Response
    ↓
gTTS
    ↓
MP3 Bytes
    ↓
Streamlit Audio
    ↓
Voice Feedback

For repeated form corrections, the voice pipeline currently applies a 5-second cooldown.

🏗️ Project Architecture

The application is organized into separate functional layers.

Main App
   │
   ├── Computer Vision
   │      └── MediaPipe + WebRTC
   │
   ├── Exercise Detectors
   │      ├── Squat
   │      ├── Push-up
   │      ├── Biceps Curl
   │      ├── Shoulder Press
   │      └── Lunges
   │
   ├── Workout Tracking
   │      └── Metrics + Session State
   │
   ├── AI Coaching
   │      ├── Groq LLM
   │      └── gTTS
   │
   └── Persistence
          └── SQLite

Application Layer

Main App/main.py

Handles Streamlit setup, login, workout planning, workout controls, camera initialization, AI/voice initialization, metrics, and workout history.

Vision Layer

services/vision/exercise_video_processor.py

Responsible for camera-frame processing, MediaPipe, pose detection, exercise selection, detector execution, skeleton rendering, form overlays, and current metrics.

Exercise Layer

detectors/

Contains the five exercise-specific movement and form detectors.

Core Layer

core/base_exercise.py

Provides shared detector functionality such as landmark handling, angle calculations, and repetition state management.

Coaching Layer

services/coaching/
├── llm.py
├── tts.py
└── voice_pipeline.py

Connects workout events to the LLM and then converts generated coaching text into speech.

Persistence Layer

services/persistence/exercise_repository.py

Handles SQLite connections, users, database initialization, workout storage, and workout history.

Configuration Layer

services/config/workout_config.py

Stores exercise choices, pose connections, default metrics, and the LLM system prompt.

Tracking Layer

services/tracking/metrics.py

Keeps the latest vision metrics synchronized with Streamlit session state.

🗄️ Workout Data

Workout information is stored using SQLite.

The persistence layer supports:

User creation

User lookup

Database initialization

Workout storage

Workout history retrieval

This allows the application to combine real-time workout analysis with historical workout tracking.

🔐 Design Approach

A key architectural choice is keeping deterministic computer-vision logic separate from generative AI.

MediaPipe
   ↓
Pose Landmarks
   ↓
Exercise Rules
   ↓
Reliable Metrics
   ↓
AI Coaching Language

This makes the LLM a coaching and communication component rather than the source of the actual repetition count or pose measurement.

📌 Key Technologies

Python

Streamlit

streamlit-webrtc

OpenCV

MediaPipe Pose Landmarker

Groq API

Llama 3.3 70B

gTTS

SQLite

📁 Important Project Structure

project/
├── Main App/
│   └── main.py
│
├── core/
│   └── base_exercise.py
│
├── detectors/
│   ├── squat.py
│   ├── pushup.py
│   ├── biceps_curl.py
│   ├── shoulder_press.py
│   └── lunges.py
│
├── services/
│   ├── vision/
│   │   └── exercise_video_processor.py
│   ├── coaching/
│   │   ├── llm.py
│   │   ├── tts.py
│   │   └── voice_pipeline.py
│   ├── persistence/
│   │   └── exercise_repository.py
│   ├── config/
│   │   └── workout_config.py
│   └── tracking/
│       └── metrics.py
│
└── ml_models/
    └── pose_landmarker_full.task

🎯 Project Goal

The goal of this project is to create a more responsive and interactive digital fitness experience where users can exercise in front of a normal camera and receive immediate technique feedback.

By combining real-time pose estimation, rule-based exercise analysis, generative AI, voice synthesis, and workout history, the system works as more than a simple rep counter—it acts as an AI-assisted virtual gym coach.

📄 Source Basis

This README is based on the project's existing technical documentation and architecture. The implementation details, exercise logic, model configuration, AI pipeline, and project structure reflect the supplied project documentation.
