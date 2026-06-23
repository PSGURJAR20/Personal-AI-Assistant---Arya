# Personal-AI-Assistant---Arya
#!/usr/bin/env python3
"""
Arya - Your Personal AI Assistant with Voice
Powered by OpenAI GPT | Voice: SpeechRecognition + pyttsx3
"""

import os, json, datetime, re, sys, threading, time

# ── Dependency checks ──────────────────────────────────────────────────────
try:
    from openai import OpenAI
except ImportError:
    print("Run: python -m pip install openai"); sys.exit(1)
try:
    import pyttsx3
except ImportError:
    print("Run: python -m pip install pyttsx3"); sys.exit(1)
try:
    import speech_recognition as sr
except ImportError:
    print("Run: python -m pip install SpeechRecognition"); sys.exit(1)

# ══════════════════════════════════════════════════════════════════════════════
# CONFIG  —  paste your key here
# ══════════════════════════════════════════════════════════════════════════════
API_KEY   = "i have not provided my key for the security resons you can use your own api key and use the Arya"
MODEL     = "gpt-3.5-turbo"
DATA_FILE = os.path.expanduser("~/.arya_data.json")
MAX_HIST  = 20

# ── Voice mode toggle (True = voice on by default) ────────────────────────
voice_enabled = True

# ══════════════════════════════════════════════════════════════════════════════
# TEXT-TO-SPEECH  (Arya speaks)
# ══════════════════════════════════════════════════════════════════════════════
tts_engine = pyttsx3.init()
tts_engine.setProperty("rate", 175)       # speed (words per minute)
tts_engine.setProperty("volume", 1.0)

# Try to pick a female voice
voices = tts_engine.getProperty("voices")
for v in voices:
    if "female" in v.name.lower() or "zira" in v.name.lower() or "hazel" in v.name.lower():
        tts_engine.setProperty("voice", v.id)
        break

def speak(text: str):
    if not voice_enabled:
        return
    # Strip ANSI colour codes before speaking
    clean = re.sub(r'\033\[[0-9;]*m', '', text)
    # Remove emojis / non-ascii symbols
    clean = clean.encode("ascii", "ignore").decode()
    tts_engine.say(clean)
    tts_engine.runAndWait()

# ══════════════════════════════════════════════════════════════════════════════
# SPEECH-TO-TEXT  (you speak)
# ══════════════════════════════════════════════════════════════════════════════
recognizer = sr.Recognizer()
recognizer.pause_threshold = 1.0   # seconds of silence before stopping

def listen() -> str:
    """Listen from microphone and return recognised text, or '' on failure."""
    with sr.Microphone() as source:
        print("\n  [Arya is listening... speak now]")
        recognizer.adjust_for_ambient_noise(source, duration=0.5)
        try:
            audio = recognizer.listen(source, timeout=6, phrase_time_limit=15)
        except sr.WaitTimeoutError:
            print("  [No speech detected]")
            return ""
    try:
        text = recognizer.recognize_google(audio)
        print(f"  [You said]: {text}")
        return text
    except sr.UnknownValueError:
        print("  [Could not understand audio]")
        return ""
    except sr.RequestError as e:
        print(f"  [Speech service error: {e}]")
        return ""

# ══════════════════════════════════════════════════════════════════════════════
# STORAGE
# ══════════════════════════════════════════════════════════════════════════════
def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE) as f: return json.load(f)
    return {"tasks": [], "notes": []}

def save_data(data):
    with open(DATA_FILE, "w") as f: json.dump(data, f, indent=2)

# ══════════════════════════════════════════════════════════════════════════════
# OPENAI
# ══════════════════════════════════════════════════════════════════════════════
client = OpenAI(api_key=API_KEY)

def build_system(data):
    now   = datetime.datetime.now().strftime("%A, %d %B %Y %H:%M")
    tasks = "\n".join(f"- {t['desc']}" for t in data["tasks"] if not t["done"]) or "None"
    notes = "\n".join(f"- {n['text']}" for n in data["notes"][-5:]) or "None"
    return f"""You are Arya, a warm, clever, and concise personal AI assistant.
Your name is Arya — never call yourself anything else.
Current date/time: {now}
User's open tasks:\n{tasks}
User's recent notes:\n{notes}
Keep replies brief and friendly. Use plain text only — no markdown."""

def ask_arya(messages, system):
    try:
        r = client.chat.completions.create(
            model=MODEL,
            messages=[{"role":"system","content":system}] + messages,
            max_tokens=1024,
            temperature=0.7,
        )
        return r.choices[0].message.content.strip()
    except Exception as e:
        return f"Sorry, I had an error: {e}"

# ══════════════════════════════════════════════════════════════════════════════
# COMMAND HANDLER
# ══════════════════════════════════════════════════════════════════════════════
HELP = """
Arya - Commands
---------------
TASKS
  add task <text>       Add a to-do item
  list tasks            Show all tasks
  done <number>         Mark task complete
  delete task <#>       Remove a task

NOTES
  add note <text>       Save a quick note
  list notes            Show all notes
  delete note <#>       Remove a note

DATE & TIME
  date / time           Current date or time

VOICE
  voice on              Enable voice mode
  voice off             Disable voice mode

SYSTEM
  history               Show chat history
  reset                 Clear conversation
  clear                 Clear screen
  help                  Show this help
  quit / exit           Exit

  Anything else -> Arya answers using GPT!
"""

def handle_local(inp, data, history):
    """Returns (reply, handled). If not handled -> send to GPT."""
    raw  = inp.strip()
    low  = raw.lower().strip()
    now  = datetime.datetime.now()
    global voice_enabled

    if low in ("help","?"): return HELP, True

    if low in ("date","today"):
        return now.strftime("%A, %d %B %Y"), True
    if low in ("time","clock"):
        return now.strftime("%H:%M:%S"), True

    if low == "voice on":
        voice_enabled = True;  return "Voice mode ON. I will now speak my replies!", True
    if low == "voice off":
        voice_enabled = False; return "Voice mode OFF. Text only from now.", True

    if low.startswith("add task "):
        desc = raw[9:].strip()
        data["tasks"].append({"desc":desc,"done":False,"added":now.isoformat()})
        save_data(data); return f"Task added: {desc}", True

    if low in ("list tasks","tasks","todo","my tasks"):
        tasks = data["tasks"]
        if not tasks: return "No tasks yet. Say: add task followed by what you need to do.", True
        lines = ["Your tasks:"]
        for i,t in enumerate(tasks,1):
            mark = "Done" if t["done"] else "Pending"
            lines.append(f"  {i}. [{mark}] {t['desc']}")
        pending = sum(1 for t in tasks if not t["done"])
        lines.append(f"  {pending} pending, {len(tasks)-pending} done.")
        return "\n".join(lines), True

    if re.match(r'^done \d+$', low):
        try:
            idx = int(raw.split()[1])-1
            data["tasks"][idx]["done"] = True
            save_data(data)
            return f"Marked as done: {data['tasks'][idx]['desc']}", True
        except: return "Invalid task number.", True

    if re.match(r'^delete task \d+$', low):
        try:
            idx = int(raw.split()[2])-1
            r = data["tasks"].pop(idx); save_data(data)
            return f"Removed task: {r['desc']}", True
        except: return "Invalid task number.", True

    if low.startswith("add note "):
        note = raw[9:].strip()
        data["notes"].append({"text":note,"added":now.isoformat()})
        save_data(data); return "Note saved!", True

    if low in ("list notes","notes","my notes"):
        notes = data["notes"]
        if not notes: return "No notes yet. Say: add note followed by your note.", True
        lines = ["Your notes:"]
        for i,n in enumerate(notes,1):
            lines.append(f"  {i}. {n['text']}")
        return "\n".join(lines), True

    if re.match(r'^delete note \d+$', low):
        try:
            idx = int(raw.split()[2])-1
            r = data["notes"].pop(idx); save_data(data)
            return f"Removed note: {r['text']}", True
        except: return "Invalid note number.", True

    if low == "history":
        if not history: return "No conversation history yet.", True
        lines = ["Recent conversation:"]
        for m in history[-6:]:
            role = "You" if m["role"]=="user" else "Arya"
            lines.append(f"  {role}: {m['content'][:80]}")
        return "\n".join(lines), True

    if low == "clear":
        os.system("cls"); return "", True

    return "", False   # not handled -> GPT

# ══════════════════════════════════════════════════════════════════════════════
# MAIN
# ══════════════════════════════════════════════════════════════════════════════
def main():
    os.system("cls")
    now = datetime.datetime.now()
    tod = "morning" if now.hour<12 else "afternoon" if now.hour<17 else "evening"

    banner = f"""
  ╔══════════════════════════════════════════╗
  ║          Hi, I'm Arya!                  ║
  ║     Your Personal AI Assistant          ║
  ║   Voice + Text | Powered by OpenAI      ║
  ╚══════════════════════════════════════════╝
  Good {tod}! {now.strftime('%A, %d %B %Y')}
  Voice mode: {'ON  (say your message or type it)' if voice_enabled else 'OFF (type your message)'}
  Type 'help' for commands | 'voice off' to disable voice | 'quit' to exit
"""
    print(banner)

    greeting = f"Good {tod}! I am Arya, your personal assistant. How can I help you today?"
    print(f"Arya: {greeting}\n")
    speak(greeting)

    data, history = load_data(), []

    pending = [t for t in data["tasks"] if not t["done"]]
    if pending:
        msg = f"You have {len(pending)} pending task. Type list tasks to see them."
        print(f"  [{msg}]\n")
        speak(msg)

    while True:
        try:
            if voice_enabled:
                print("  [Press ENTER to type  |  Just speak to use voice]")
                # Non-blocking: check if user pressed Enter (to type) or just start listening
                print("You (speak or type): ", end="", flush=True)

                # Run listener in thread; if user types something cancel voice
                result_holder = [""]
                stop_flag     = threading.Event()

                def voice_thread():
                    text = listen()
                    if not stop_flag.is_set():
                        result_holder[0] = text

                t = threading.Thread(target=voice_thread, daemon=True)
                t.start()

                # Wait briefly then check if user is typing
                typed = input()   # blocks until Enter
                stop_flag.set()

                if typed.strip():
                    user_input = typed.strip()
                else:
                    t.join(timeout=10)
                    user_input = result_holder[0].strip()
                    if not user_input:
                        continue
            else:
                user_input = input("You: ").strip()
                if not user_input:
                    continue

        except (EOFError, KeyboardInterrupt):
            bye = "Goodbye! Have a great day!"
            print(f"\nArya: {bye}")
            speak(bye)
            break

        if user_input.lower() in ("quit","exit","bye","goodbye"):
            bye = "Goodbye! Take care!"
            print(f"Arya: {bye}")
            speak(bye)
            break

        if user_input.lower() == "reset":
            history.clear()
            msg = "Conversation cleared. Your tasks and notes are safe."
            print(f"Arya: {msg}\n")
            speak(msg)
            continue

        reply, handled = handle_local(user_input, data, history)
        if handled:
            if reply:
                print(f"\nArya: {reply}\n")
                speak(reply)
            continue

        # Send to GPT
        history.append({"role":"user","content":user_input})
        print("Arya: thinking...", end="\r")
        reply = ask_arya(history[-MAX_HIST:], build_system(data))
        print(" " * 20, end="\r")
        history.append({"role":"assistant","content":reply})
        if len(history) > MAX_HIST: history = history[-MAX_HIST:]
        print(f"Arya: {reply}\n")
        speak(reply)

if __name__ == "__main__":
    main()
