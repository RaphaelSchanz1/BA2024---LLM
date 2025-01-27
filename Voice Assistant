import pyaudio
import time
import wave
import keyboard
import os
import requests
import json
import promptEngineering
import re
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes
os.environ["KMP_DUPLICATE_LIB_OK"] = "TRUE"
from faster_whisper import WhisperModel

class FunctionTimer:
    def __init__(self):
        self.start_time = None
        self.end_time = None
        self.is_running = False

    def start_timer(self):
        """Starts the timer when the first function is called."""
        if not self.is_running:
            self.start_time = time.time()
            self.is_running = True
            print("Timer started.")

    def stop_timer(self):
        """Stops the timer when the last function finishes."""
        if self.is_running:
            self.end_time = time.time()
            self.is_running = False
            elapsed_time = self.end_time - self.start_time
            print(f"Timer stopped. Total elapsed time: {elapsed_time:.2f} seconds.")
            return elapsed_time

# Set up parameters
CHUNK = 1024  # Number of frames per buffer
FORMAT = pyaudio.paInt16  # Audio format
CHANNELS = 1  # Number of audio channels (1 for mono)
RATE = 16000  # Sampling rate (samples per second)
WAV_OUTPUT_FILENAME = "temp_recording.wav"  # Temporary file to store audio
languageForQuery= ""

# Function to record audio
def record_audio():
    p = pyaudio.PyAudio()

    stream = p.open(format=FORMAT,                                                                                                                          
                    channels=CHANNELS,
                    rate=RATE,
                    input=True,
                    frames_per_buffer=CHUNK)

    print("Recording... Press 'space' to stop.")

    frames = []

    # Start recording when space key is pressed, stop when it's released
    try:
        while keyboard.is_pressed('space'):  # While the space key is held down
            data = stream.read(CHUNK)
            frames.append(data)

    except KeyboardInterrupt:
        pass

    print("Recording stopped.")

    # Stop and close the stream
    stream.stop_stream()
    stream.close()
    p.terminate()

    # Save recorded audio to a .wav file
    wf = wave.open(WAV_OUTPUT_FILENAME, 'wb')
    wf.setnchannels(CHANNELS)
    wf.setsampwidth(p.get_sample_size(FORMAT))
    wf.setframerate(RATE)
    wf.writeframes(b''.join(frames))
    wf.close()

    print(f"Audio saved to {WAV_OUTPUT_FILENAME}")

    # Pass the audio file to the transcription function
    process_audio_for_transcription(WAV_OUTPUT_FILENAME)

    if os.path.exists(WAV_OUTPUT_FILENAME):
        os.remove(WAV_OUTPUT_FILENAME)
        print(f"Deleted the temporary audio file: {WAV_OUTPUT_FILENAME}")

# Function to process audio for transcription and then send transcription as prompt
def process_audio_for_transcription(audio_file):

    ## Should not crash of audio recording is empty##
    """
    This function will take the recorded audio file, transcribe it, 
    and send the transcription as a prompt to the ollama API.
    """
    print(f"Processing audio file for transcription: {audio_file}")

    model_size = "medium"                                                                                                                                                                                      

    # Run on CPU with INT8
    model = WhisperModel(model_size, device="cpu", compute_type="int8")

    segments, info = model.transcribe(audio_file, beam_size=5)

    print("Detected language '%s' with probability %f" % (info.language, info.language_probability))

    #set global language of user
    global languageForQuery                                                                                                                                                                                                                                                                                                                                                                                                                                   
    languageForQuery = info.language

    transcription = ""

    for segment in segments:
        transcription += segment.text + " "
        print("[%.2fs -> %.2fs] %s" % (segment.start, segment.end, segment.text))

    # Send transcription to the API as a prompt
    if transcription.strip():  # Check if there's any transcription to send
        print("Sending transcription to API...")
        send_prompt_to_api(transcription.strip(),info.language)
    else:
        print("No transcription to send.")

# Function to send the transcription as a prompt to ollama API
def send_prompt_to_api(transcription, language):    

    userData = ". the following is previously collected userData: "+readUserData()       

    prePrompt = readPrePrompt()                                                                                                                                                                                                      
    
    headers = {
        'Content-Type': 'application/json',
    }

    #ata = {
    #    "model": "llama3.1",
    #    "prompt": promptEngineering.prePromptVerbose+transcription+"Respond in the following language:"+language  # Using the transcription as the prompt
    #}

    ##8b
    data8b = {
        "model": "llama3.1",
        "prompt": promptEngineering.prePromptshortNoLines+transcription+"Respond in the following language:"+language  # Using the transcription as the prompt
    }

    data8bTemperature = {
    "model": "llama3.1",
    "prompt": promptEngineering.prePromptshortNoLines+transcription+"Respond in the following language:"+language,
    "options": {
        "temperature": 0.7  # Adjust temperature as needed  # Using the transcription as the prompt
        }
    }

    ##1b
    data1b = {
    "model": "llama3.2:1b",
    "prompt": promptEngineering.prePromptshort+transcription+"Respond in the following language:"+language  # Using the transcription as the prompt
    }

    dataPhi3b = {
    "model": "phi3:3.8b",
    "prompt": promptEngineering.prePromptshortNoLines+transcription+"Respond in the following language:"+language  # Using the transcription as the prompt
    }

    dataPhi3bTemperature = {
    "model": "phi3:3.8b",
    "options": {
        "temperature": 0,                                                                                                                                                                                                                                                                                                                                                            # Adjust temperature as needed
     },
    "prompt": promptEngineering.prePromptshortNoLines + transcription + "Respond in the following language:" + language
    }
                                                                                                                                                                                                                                                                     
    modelused = data8b


    #data = {
    #"model": "llama3.1",                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        
    #"prompt": str(prePrompt)+" "+transcription+" #endregion "+"Respond in the following language:"+language+userData  # Using the transcription as the prompt
    #}

    timingPrompt("Prompt: "+str(modelused)+" ")
    #print(promptEngineering.prePromptVerbose+transcription+"Respond in the following language:"+language)                                                                                                                                                                                                                                                                                                                                                                                                                                                                            
    #print(str(prePrompt)+transcription+"endregion "+"Respond in the following language:"+language+userData)
    response = requests.post('http://localhost:11434/api/generate', headers=headers, json=modelused)

    # Print the API response
    responseConcat = ""
    # Convert JSON string to Python dictionary
    ##data = json.loads(response.text)

    # Access specific attributes
    json_lines = response.text.strip().split('\n')

    # Parse each line individually
    for line in json_lines:
        responseData = json.loads(line)
        responseConcat = responseConcat + responseData['response']
        #print(f"Model: {responseData['model']}, Response: {responseData['response']}")
        #print("Response over")

    timingPrompt("Response: "+responseConcat+" ")
    parseAndCall(extract_json_from_string(responseConcat))


def readUserData():

    filename = "C:\\Users\\rafae\\Desktop\\BA_20240414\\CodeBase\\Voice Assistant\\transcribeAndResponse\\userData.txt"

    try:
        with open(filename, 'r') as file:  # Open the file in read mode
            content = file.read()  # Read the entire content of the file
        #return " Only consider if userData is relevant to the request:"+content
        return " The following is previously collected USER DATA. Only consider the USER DATA when its relevant to the conversation and dont add any artifical information: " +content
    except FileNotFoundError:
        return "Error: File not found."
    except Exception as e:
        return f"An error occurred: {e}"
    
def readPrePrompt():

    filename = "C:\\Users\\rafae\\Desktop\\BA_20240414\\CodeBase\\Voice Assistant\\transcribeAndResponse\\prePromptTest.txt"

    try:
        with open(filename, 'r') as file:  # Open the file in read mode
            content = file.read()  # Read the entire content of the file
        return content
    except FileNotFoundError:
        return "Error: File not found."
    except Exception as e:
        return f"An error occurred: {e}"
    

#Removes newline characters and replaces multiple spaces with a single space in the input string.
def clean_string(input_string: str) -> str:

    # Remove newline characters
    cleaned_string = input_string.replace('\n', ' ')
    # Replace multiple spaces with a single space using regular expressions
    cleaned_string = re.sub(r'\s+', ' ', cleaned_string)
    # Strip leading and trailing spaces
    return cleaned_string.strip()

#check if erroneous string provided
def extract_json_from_string(input_string):

    try:
        # Find the first occurrence of '{' and last occurrence of '}' in the string
        start_index = input_string.find('{')
        end_index = input_string.rfind('}')
        
        # If both indices are valid
        if start_index != -1 and end_index != -1:
            # Extract the substring that should contain the JSON
            json_string = input_string[start_index:end_index+1]
            
            # Parse the extracted substring as JSON
            json_data = json.loads(json_string)
            return json_data
        else:
            print("No valid JSON was found")
            return None  # Return None if no valid JSON found
        
    except json.JSONDecodeError as e:
        print("No valid JSON was found")
        print(f"Initial JSON error: {e}")
        corrected_string = json_string
        
        # Check for lists in functions and ensure they are properly closed
        if 'list' in corrected_string:
            print("first if")
            # Find the list section and ensure it ends with a closing bracket
            list_start = corrected_string.find('"list": [') + len('"list": ["')
            list_end = corrected_string.find(']', list_start)
            if list_end == -1:  # No closing bracket found
                corrected_string = corrected_string.rstrip() + "]"  # Add closing bracket
        print("new JSON: ")
        print(corrected_string)

        # Optionally, you could implement more specific corrections here.
        
        # Now, try to parse again after modification
        try:
            json_object = json.loads(corrected_string)
            print("JSON corrected successfully!")
            return json.dumps(json_object, ensure_ascii=False, indent=2)  # Return formatted JSON
        except json.JSONDecodeError as e:
            return f"Could not correct the JSON. Error: {e}"

def parseAndCall(json_data):

    #check if correct JSON
    if isinstance(json_data, str):
        try:
            json_data = json.loads(json_data)
        except json.JSONDecodeError as e:
            print(f"Error parsing JSON: {e}")
            return
        
    functions_list = json_data["functions"]

    if "response" in json_data:
        respond(json_data['response'])

    params={}

    for function in functions_list:
        #Error Handling if JSON is formatted differently
        if 'functionName' in function:
            function_name = function['functionName']
            parameters = function['parameters']

            
            for param_key, param_value in parameters.items():

                params[param_key] = param_value
        else:         
            function_name = list(function.keys())[0]
            params = function[function_name]
            print("ELSE function Name:: "+ str(function_name))
            print("ELSE PARAMS:: "+ str(params))

        #globals() returns a dictionary with every global variable, here functions
        func = globals().get(str(function_name).lower())
        # Check if the function exists and is callable
        if callable(func):
            func(params)  # Unpack arguments if they exist
        else:
            print(f"Function '{function_name}' is not defined.")

# userdata collection
def userdata(userData):

    print("THIS IS USERDATA:: "+str(userData))
    filename = "C:\\Users\\rafae\\Desktop\\BA_20240414\\CodeBase\\Voice Assistant\\transcribeAndResponse\\userData.txt" 

    try:
        with open(filename, 'a') as file:
            file.write(str(userData) + "\n")
            file.flush()
        print(f"{userData} successfully saved to {filename}.")
    except Exception as e:
        print(f"An error occurred while saving the text: {e}")

def shoppinglist(shoppinglist):

    print("THIS IS shoppinglist:: "+str(shoppinglist))
    filename = "C:\\Users\\rafae\\Desktop\\BA_20240414\\CodeBase\\Voice Assistant\\transcribeAndResponse\\shoppinglist.txt" 

    try:
        with open(filename, 'a') as file:
            file.write(str(shoppinglist) + "\n")
            file.flush()
        print(f"Text successfully saved to {filename}.")
    except Exception as e:
        print(f"An error occurred while saving the text: {e}")

def respond(response):                                                                                                                                                                                                                                                                                                                                                                                                                            
    print("RESPOND:: "+ response)

def neighbourhoodchat(message):

    if 'Parameters' in message:
        message = message['Parameters']
    elif 'parameters' in message:
        message = message['parameters']

    if 'message' in message:
        message = message['message']
    elif 'Message' in message:
        message = message['Message']

    chat_id = ""

    token = ""
    url = f"https://api.telegram.org/bot{token}/sendMessage"
    payload = {
        'chat_id': chat_id,
        'text': message
    }

    try:
        response = requests.post(url, data=payload)
        if response.status_code == 200:
            print(f"Message: '{message}' sent successfully to chat_id {chat_id}")
        else:
            print(f"Failed to send message. Error: {response.status_code}, {response.text}")
    except Exception as e:
        print(f"An error occurred: {e}")

    print("Neighbourhoodchat:: "+ str(message))

def call(who):
    print("Call:: "+str(who))

def sendmessage(params):
    #check for capitalization
    who =""

    if 'who' in params:
        who = params['who']
    elif 'Who' in params:
        who = params['Who']
    elif 'Contacts' in params:
        who = params['Contacts']
    elif 'contacts' in params:
        who = params['contacts']
    else:
        print("ELSE NO CONTACTS:: ")

    if 'message' in params:
        message = params['message']
    elif 'Message' in params:
        message = params['Message']
    else:
        print("ELSE NO MESSAGE:: ")

    contacts = {
    "raphael": "1876914457",
    "neighbourHood": "-4596195001",
    "david": "1876914457",
    "meyer": "1876914457",
}
    
    if isinstance(who, str):
        who_lower = who.lower()  # Apply lower() if it's a string
    elif isinstance(who, list):
        # Apply lower() to each element in the list if it's a list of strings
        who_lower = [w.lower() for w in who if isinstance(w, str)]
        who_lower = who_lower[0]

    # Search for the contact in the dictionary
    chat_id = None
    if str(who_lower) in contacts:
        chat_id = contacts[who_lower]

    token = ""
    url = f"https://api.telegram.org/bot{token}/sendMessage"
    payload = {
        'chat_id': chat_id,
        'text': message
    }

    try:
        response = requests.post(url, data=payload)
        if response.status_code == 200:
            print(f"Message: '{message}' sent successfully to chat_id {chat_id}")
        else:
            print(f"Failed to send message. Error: {response.status_code}, {response.text}")
    except Exception as e:
        print(f"An error occurred: {e}")

# Replace 'YOUR_BOT_TOKEN' with the token you received from BotFather
# Replace 'CHAT_ID' with the ID of the recipient or their Telegram username with '@'


def weather(location):
    api_key = 'b15a74604062361ffc83335a49681a96'

    # check for capitalization
    if 'Parameters' in location:
        location = location['Parameters']
    elif 'parameters' in location:
        location = location['parameters']


    if 'Location' in location:
        city = location['Location']
    elif 'location' in location:
        city = location['location']
    else:
        city = str(location)

    #print("LOCATION inf FUNC:: "+location)
    #print("location['Location']:: "+str(location['Location']))
                                                                                                                                                                                                                                                                                                                                                                                             
    url = f'http://api.openweathermap.org/data/2.5/weather?q={city}&appid={api_key}&lang={languageForQuery}'

    response = requests.get(url)

    if response.status_code == 200:
        data = response.json()
        print(str(data))
        temp = data['main']['temp']
        #Kelvin to Celsius
        temp = round(temp - 273.15, 3)
        desc = data['weather'][0]['description']
        print(f'Temperature: {temp} C')
        print(f'Description: {desc}')
    else:
        print('Error fetching weather data')

def googling(query):
    
    api_key = '7bd4da105eaa289d94a853ce5dda4b7ec1c9bb01d814aa2ea26c1de94886fd2f'
    queryG = query['query']
    language = languageForQuery
    print("language for query::= "+language)

    params = {
        'engine': 'google',
        'q': queryG,
        "location": "Frankfurt, Hessen, Germany",
        'google_domain': 'google.de',
        'hl': language,
        'api_key': api_key,
    }

    response = requests.get('https://serpapi.com/search', params=params)
    data = response.json()
    knowledge_graph =""                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  

    # Extract knowledge graph information

    if 'knowledge_graph' in data:
        knowledge_graph = data['knowledge_graph']['description']
        print(knowledge_graph)
    else:
        print("Knowledge graph not found in the response.")

    if 'organic_results' in data: 
        knowledge_graph = knowledge_graph +" "+ data['organic_results'][0]['snippet']
        print(knowledge_graph)
    else:
        print("organic_results not found in the response.")

    if 'answer_box' in data: 
        knowledge_graph = knowledge_graph +" "+ data['answer_box'][0]['snippet']
        print(knowledge_graph)
    else:
        print("answer_box not found in the response.")


# Timing function called each time a user request is handled
def timingPrompt(data):

    filename = "C:\\Users\\rafae\\Desktop\\BA_20240414\\CodeBase\\Voice Assistant\\transcribeAndResponse\\timingPrompts.txt" 

    try:
        with open(filename, 'a') as file:
            file.write(str(data) + "\n")
            file.flush()
    except Exception as e:
        print(f"An error occurred while saving the text: {e}")

# Main function to trigger recording
def main():

    print("Press and hold the 'space' key to start recording...")
    keyboard.wait('space')  # Wait for the user to press 'space'
    timer = FunctionTimer()
    timer.start_timer()
    record_audio()
    time = timer.stop_timer()
    timingPrompt("Time: "+str(time)+"\n")
    #testString = 
    #parseAndCall(testString)

if __name__ == "__main__":

    main()
