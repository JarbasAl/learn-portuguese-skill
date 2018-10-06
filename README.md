# All about intents

So i hear you want to make a skill but are lost?

There's lots of words going around, intents, adapt, padatious, context, converse, voc files... it's easy to get intimidated at first, but fear no more, I'm here to help!

## IF…THEN statements?

You just gave mycroft an order, how does it decide what to do?

It helps to think about it in terms of *if.. else ... *statements

	if user said "hello" then say "hello"
	if user said "thank you" then say "you are welcome"
	
When starting a skill the first thing you should do is think of these if...else.. statements.

You don't need to write any of this down, but it helps to have a list with the bare minimum of these you need for the functionality you are trying to implement, do not worry about how the user will ask the question yet, just keep one statement for each possible outcome

What is an intent? each possible outcome of the conversation you are modeling is an intent, thats why i suggestd you write those if..else.. statements, each of those is an intent you need to teach mycroft!

## Padawhat?

Do we have to code if else statements? Hell no, we are not in the 80's, there's real AI now!

The simplest way to get started is to use padatious, you just give it examples and it knows what to do. 

You do not need to understand the next paragraphs at all to use padatious, but if you are curious of the inner working read on

Padatious is mycroft's neural network intent parser, it will ensure exact matches to any example phrase you give it by using it's old fashioned regex cousin padaos. But it also manages to understand sentences you did not explicly teach it.

Each padatious intent is a neural network model that tells you if for a piece f text you should select that intent, with a probability. Padatious engine chooses the highest scoring intent, or None if all probabilities are bellow a threshold

Specifically, it uses one-hot encoding via local vocabularies (and a few extras like an unknown ratio input) into a series of shallow feed forward networks to perform intent recognition and entity extraction.

If you didn't understand anyhing that's ok, using it in pratice is super easy!

There is a command line tool named [msk](https://github.com/MycroftAI/mycroft-skills-kit) that creates the skill template for you, you can even create complete skills if they are really simple, you can learn more about it [here](https://geekedoutsolutions.com/mycroft-skill-creation-lifecycle-using-msk) 

![](https://camo.githubusercontent.com/f25bcaae321ea183df513c4e11be3d211047de1e/68747470733a2f2f696d61676573322e696d67626f782e636f6d2f61622f32352f366b62714b6258685f6f2e676966) 

Let's make a real skill, LearnPortugueseSkill,

	if user said "say hello in portuguese" then say "olá"
	
how many ways can the user give that order?

either use msk or create the files manually

	vocab/en-us/hello_in_portuguese.intent

the file should contain all sample sentences you can think off for this intent

	say hello in portuguese
	how do you say hello in portuguese
	what is the portuguese word for hello
	translate hello to portuguese
	
Now lets look at our skill code, open the ``__ init __.py`` file


	from mycroft.skills.core import MycroftSkill, intent_file_handler

	class LearnPortugueseSkill(MycroftSkill):
	
	    @intent_file_handler("hello_in_portuguese.intent")
	    def handle_hello(self, message):
		    self.speak_dialog("hello_in_portuguese")
	    

	def create_skill():
	    return LearnPortugueseSkill()


Since this is a teach portuguese skill we need to implement some logic,

Mycroft's TTS system will be configured for english, it can not speak portuguese text, the awesome [translate skill ](https://github.com/jcasoft/translate-skill) from jcasoft uses google TTS for this, let's use his code to add a ``say_in_portuguese(text)`` method!
	
We will be using this method to speak the portuguese bits


	from mycroft.skills.core import MycroftSkill, intent_file_handler
	from mycroft.audio import wait_while_speaking, play_mp3
	import os


	class LearnPortugueseSkill(MycroftSkill):
	    path_translated_file = "/tmp/portuguese.mp3"

        def translate_to_portuguese(self, text):
            translated = translate(text, "pt")
            self.speak_portuguese(translated)

        def speak_portuguese(self, sentence):
            wait_while_speaking()
            get_sentence = 'wget -q -U Mozilla -O ' + self.path_translated_file + \
                           '"https://translate.google.com/translate_tts?ie=UTF-8&tl=pt&q=' + \
                           str(sentence) + '&client=tw-ob' + '"'
    
            os.system(get_sentence)
            play_mp3(self.path_translated_file)

	    @intent_file_handler("hello_in_portuguese.intent")
	    def handle_hello(self, message):
            # wait=True makes mycroft wait until speech is finished playing before continuing
            self.speak_dialog("hello_in_portuguese", wait=True)
            self.speak_portuguese("olá")

notice that ``self.speak_dialog("hello_in_portuguese")`` ? That line of code is using a dialog file, msk/you should have created it at
	
	dialog/en-us/hello_in_portuguese.dialog

the file should contain all sample sentences mycroft can speak to the user, one will be picked randomly

	hello in portuguese is 
	you say it like this

To add support for new languages just create a new folder with it's language code, you can edit your examples at any time without touching the code, the skill will even auto reload, simple isn't it?

It is also much easier to translate an intent if you have full sentences, adapt is trickier and often requires the translator to check the source code, one more good reason to use padatious!

NOTE mycroft recently changed this, all files are in ``skill_folder/locale/en-us`` , old way still works, but now its easier since you need to track even less folders!

## I like Rules


	if user said "say thank you in portuguese" then say "obrigado" and explain gender
	
If you are like me it is easier for you to think of rules than to come up with examples, or maybe you just need more control over when the intent triggers

Adapt comes to the rescue here, instead of magically learning from examples you can specify some rules for when you want it to trigger.

	    @intent_handler(IntentBuilder("ThankYouIntent")
	                    .require("ThankYou")
	                    .require("inPortuguese"))
        def handle_thank_you(self, message):
            self.speak_portuguese("obrigado")


For this you need to create a file with keyword examples

        # ThankYou.voc
        thank you
        thanks
        thx

        # inPortuguese.voc
        in portuguese
        in portugal
        in pt
        in p t

## Extracting Keywords
	

Every keyword you defined for adapt can be used as a variable in the intent code, let's improve our adapt intent to use a gender keyword

	    @intent_handler(IntentBuilder("ThankYouIntent")
	                    .require("ThankYou")
                        .require("inPortuguese")
                        .optionally("gender"))
        def handle_thank_you(self, message):
            gender = message.data.get("gender")
            if not gender:
                self.speak_dialog("if_male", wait=True)
                self.speak_portuguese("obrigado")
            elif gender == "male":
                self.speak_portuguese("obrigado")
            elif gender == "female":
                self.speak_portuguese("obrigada")
                
This is the main use case for optional keywords, but they can also be used to increase confidence and help in disambiguation, after having the bare minimum requirements feel free to add as much optional keywords as you want

        @intent_handler(IntentBuilder("ThankYouIntent")
                .optionally("how")
                .require("ThankYou")
                .require("inPortuguese")
                .optionally("gender"))
        def handle_thank_you(self, message):
            ...
                
But what if we want to extract something from text that we don't have an example of? 
Padatious makes it super simple, in your .intent file write your examples like this

	say {{words}} in portuguese
    speak {{words}} in portuguese
    translate {{words}} to portuguese
	
that's it, you can now access it just like with adapt

	    @intent_file_handler("say.intent")
        def handle_say(self, message):
            words = message.data["words"]
            self.speak_portuguese(words)
	

## What about multi turn dialog?

I have written a [blog post](https://jarbasal.github.io/posts/2018/10/skill_guidelines_2/) about this before, so i don't want to repeat myself a lot

adapt intent parser supports contexts, a context is a keyword that becomes available even if the user didnt say it

This can be used for continuous dialog, you can provide data with it (the missing keyword value) 


	if user said "say again" then repeat last portuguese sentence
	
	
Let's implement repeat functionality, first create repeat.voc

    say that again
    repeat

Now let's change the translate_to_portuguese method to set a context

        def translate_to_portuguese(self, text):
            translated = translate(text, "pt")
            self.speak_portuguese(translated)
            self.set_context("previous_speech", translated)

            
The context made the "previous_speech" keyword available to adapt, this intent can now be triggered up to 3 questions after say_in_portuguese was last triggered

        @intent_handler(IntentBuilder("RepeatIntent")
                        .require("repeat")
                        .require("previous_speech"))
        def handle_repeat(self, message):
            text = message.data.get("previous_speech")
            self.say_in_portuguese(text)
         

Padatious does not yet support context, for these cases you are stuck with adapt, however you can set and remove contexts at will inside padatious intents like i just did in the speak_portuguese method

You can also use contexts that don't even have data, you just require it to ensure that something happened first

       
        def speak_portuguese(self, sentence):
            wait_while_speaking()
            get_sentence = 'wget -q -U Mozilla -O ' + self.path_translated_file + \
                           '"https://translate.google.com/translate_tts?ie=UTF-8&tl=pt&q=' + \
                           str(sentence) + '&client=tw-ob' + '"'
    
            os.system(get_sentence)
            play_mp3(self.path_translated_file)
            self.set_context("google_tx")
    	
        @intent_handler(IntentBuilder("HowDoYouKnowIntent")
                        .require("question").require("know")
                        .require("google_tx"))
        def handle_how_do_you_know(self, message):
            self.speak_dialog("google_told_me")


## I am a wizard

So i hear you like regex and padatious is not an option for you, or maybe you are afraid because you don't understand how padatious learns, but you aren't the botphobic kind are you?

	if user said "explain {SampleWord} in portuguese" then say dictionary definition in portuguese

For this i will be using [PyDictionary](https://pypi.org/project/PyDictionary/), you can add it to your skill by creating a requirements.txt file and adding

    PyDictionary
    	
I always recommend you use padatious, but sometimes regex is a necessary evil, [Pythex](https://pythex.org/) is useful to check your regex

Make a file named dictionary.rx

	explain (?P<SampleWord>.*) in portuguese
    meaning of (?P<SampleWord>.*) in portuguese

Just require and use the regex capture group as a normal adapt keyword

        @intent_handler(IntentBuilder("ExplainInPortugueseIntent")
                    .require("SampleWord"))
        def handle_explain(self, message):
            word = message.data["SampleWord"]
            dictionary = PyDictionary()
            dictionary.meaning(word)
            meaning = dictionary.get("Noun") or dictionary.get("Verb")
            self.translate_to_portuguese(meaning)
        
Padatious is much more human readable, easier to translate, and less prone to errors, adapt's regex is also known to be somewhat buggy at times, but maybe you really are a wizard. 
Just keep in mind adapt is rule based, corner cases creep up if you over simplify/complicate

	"hey mycroft, where is the nearest bus stop"
	"stopping everything"

	"hey mycroft, do i need to say the last order again?
	"the last order again"
	

## I am not a wizard!

ok, you are still having trouble, there's some corner case you can't figure, need to extract a date or a number maybe?

    if user said "say (weekday) date in portuguese" then translate date to portuguese
    if user said "say number {number} in portuguese" then translate number to portuguese
        
We have utility packages to extract dates, english is well supported

        @intent_handler(IntentBuilder("DateInPortugueseIntent")
                        .require("inPortuguese").require("date").require("say"))
        def handle_date(self, message):
            date, text_remainder = extract_datetime(message.data["utterance"], lang=self.lang)
            pronounced_date = nice_date(date)
            self.translate_to_portuguese(pronounced_date)

One useful strategy that works well together with optional keywords is to use the utterance_remainder, in adapt intents you can get the text leftover that was not captured by the intent

Mycroft also provides utils to handle numbers, language support not guaranteed except for english.

The PR for portuguese is in, we can pronounce the number directly and save a call to google translate
  
        @intent_handler(IntentBuilder("NumberInPortugueseIntent")
                        .require("inPortuguese").require("number").require("say"))
        def handle_number(self, message):
            text = message.utterance_remainder()
            # lets get a number from the utterance
            number = extract_number(text, lang=self.lang)
            # portuguese uses long scale, lets take that into account!
            # in short scale 1 billion = 1e12 instead of 1e9
            spoken_number = pronounce_number(number, lang="pt", short_scale=False)
            self.speak_in_portuguese(spoken_number)
            
## No intents needed

Can you answer some general purpose question? Mycroft also has a mechanism to try to answer things it doesn't have intents for

Fallback skills are tried by order until one can answer you, this is a good place to plug skills like wolfram alpha

But sometimes you just could not make an intent for your use case, maybe because there are a lot of skills and there are intent colisions

    if user said "say a pun in portuguese" then say a pun only portuguese speakers can understand
    
A last resort thing you can do is make a fallback intent, use good old fashioned python programming with no help whatsoever to decide what to do

        class LearnPortugueseSkill(FallbackSkill):
            path_translated_file = "/tmp/portuguese.mp3"
            intercepting = False
            puns = [...]
            
            def initialize(self):
                self.register_fallback(self.handle_pun_fallback, 99)
                
There's usually no good reason to do this, and this example should have used a regular intent

	    def handle_pun_fallback(self, utterance):
            if self.voc_match(self, utterance, "pt_pun"):
                pun = random.choice(self.puns)
                question = pun["pergunta"]
                answer = pun["resposta"]
                self.speak_portuguese(question + ".\n" + answer)
                return True
            return False
    
        
## Intercepting the intent parser

What if we want to capture the whole utterance cycle inside a skill without giving other skills a chance?

     if user said "translate everything i say to portuguese" then start ignoring questions and translate everything
     
Let's make an intent to start the interception cycle

        @intent_file_handler("live_translate_portuguese.intent")
        def handle_live_translate(self, message):
            self.speak_dialog("start_tx", wait=True)
            self.speak_portuguese("iniciando tradução automática")
            self.intercepting = True
    
Now we can use the converse method to capture the utterances

We also need to check if the user told us to stop, there is an helper method to check if some voc file would match
            
        def stop(self):
            if self.intercepting:
                self.speak_dialog("stop_tx", wait=True)
                self.speak_portuguese("parando tradução automática")
                self.intercepting = False
                
        def converse(self, transcript, lang="en-us"):
            utterance = transcript[0]
            if self.intercepting:
                if self.voc_match(self, utterance, "cancel", lang=lang):
                    self.stop()
                else:
                    self.speak_portuguese(utterance)
                return True
            return False

Now every time you say "Hey mycroft" it will repeat what you said in portuguese and ignore any questions

If you stop interacting with mycroft for 5 minutes converse method will no longer be called, let's set a timer to reset our state variable in case user abandons midway

    TODO code

## So what did we learn?

I wanted to show you the potential of mycroft intent system, some functionality of this skill could probably be made other (better) ways

Hopefully mycroft is less confusing for you know, this example skill should teach you to

- speak pre programmed answers, in our case translate hello to portuguese
- optionally take extra parameters into account (gender matters when speaking in portuguese)
- extract keywords with padatious and translate them to portuguese 
- extract keywords with regex and explain their meaning in portuguese 
- ask follow up questions (how does mycroft know portuguese)
- pass data to follow up intents (repeat last portuguese utterance)
- use mycroft helper utils, utterance_remainder, extract/pronounce number and extract/pronounce date
- create a fallback handler and use custom utterance parsing
- take control of the utterance processing cycle using converse method (live translate anything to portuguese)

Source code is available [here]()

## Only for mycroft?

Adapt and padatious are open source, you can use them outside mycroft in your own projects! In fact, [mozilla is using adapt](https://mycroft.ai/blog/myzilla-mycroft-at-the-mozilla-all-hands/#project-things) 

Installing padatious is easy

	sudo apt-get install libfann-dev python3-dev python3-pip swig
	pip install padatious
	
Installing adapt is also easy

	pip install -e git+https://github.com/mycroftai/adapt#egg=adapt-parser

Padatious is really simple to use

	from padatious import IntentContainer

	container = IntentContainer('intent_cache')
	container.add_intent('hello', ['Hi there!', 'Hello.'])
	container.add_intent('goodbye', ['See you!', 'Goodbye!'])
	container.add_intent('search', ['Search for {query} (using|on) {engine}.'])
	container.train()

	print(container.calc_intent('Hello there!'))
	print(container.calc_intent('Search for cats on CatTube.'))

	container.remove_intent('goodbye')

Adapt seems a little more intimidating at first sight

	from adapt.intent import IntentBuilder
	from adapt.engine import IntentDeterminationEngine

	engine = IntentDeterminationEngine()

	weather_keyword = [
	    "weather"
	]

	for wk in weather_keyword:
	    engine.register_entity(wk, "WeatherKeyword")

	weather_types = [
	    "snow",
	    "rain",
	    "wind",
	    "sleet",
	    "sun"
	]

	for wt in weather_types:
	    engine.register_entity(wt, "WeatherType")

	locations = [
	    "Seattle",
	    "San Francisco",
	    "Tokyo"
	]

	for loc in locations:
	    engine.register_entity(loc, "Location")

	weather_intent = IntentBuilder("WeatherIntent")\
	    .require("WeatherKeyword")\
	    .optionally("WeatherType")\
	    .require("Location")\
	    .build()

	engine.register_intent_parser(weather_intent)

	for intent in engine.determine_intent(' '.join(sys.argv[1:])):
		if intent.get('confidence') > 0:
		print(json.dumps(intent, indent=4))

You can also get [Padaos](https://github.com/MatthewScholefield/padaos)  the regex intent parser, i even[ forked it](https://github.com/JarbasAl/auto_regex) it to simply output the generated regex instead of making intents

	from padaos import IntentContainer

	container = IntentContainer()
	container.add_intent('hello', [
	    'hello', 'hi', 'how are you', "what's up"
	])
	container.add_intent('buy', [
	    'buy {item}', 'purchase {item}', 'get {item}', 'get {item} for me'
	])
	container.add_intent('search', [
	    'search for {query} on {engine}', 'using {engine} (search|look) for {query}',
	    'find {query} (with|using) {engine}'
	])
	container.add_entity('engine', ['abc', 'xyz'])
	container.calc_intent('find cats using xyz')
	# {'name': 'search', 'entities': {'query': 'cats', 'engine': 'xyz'}}
