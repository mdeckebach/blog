.. title: Lessons Learned from Making Call Center Calculator
.. slug: lessons-learned-from-making-call-center-calculator
.. date: 2022-05-29 18:25:21 UTC-07:00
.. tags: Data
.. category: 
.. link: 
.. description: 
.. type: text



Earlier this spring, I built a web app to show the differences between Erlang C and computer simulation methods for determining call center staffing requirements. Nerdy, I know.

It was a fun project that combined much of my work at Patagonia over the past couple years (call center staffing) with some of my CS302 coursework (event-driven simulations).

Originally, I started the project in Python using Flask, but quickly realized that I could do everything I wanted client-side with JavaScript. Toss in some static HTML and CSS and you have a functioning web app. Pretty neat!

That's not to say that everything was roses and sunshine. Below are the three biggest hurdles I encountered working on this project, and what I learned as a result.

1. Event Class Overwriting
==========================

When I first started tinkering with the idea of simulating a call center queue, I had no idea what I was doing. I created a naive loop like this, where each iteration represented one second in the time period:

.. code-block:: JavaScript
    :linenos:

    for second in total_seconds:
        if second in contact_start_times:
            make new contact + do stuff
        for contact in active_contacts:
            if contact.end_time = second
                do stuff

While it worked, it wasn't efficient at all. Given 100 simulations per agent count, ~8 agent counts to test, 50 contacts, and a period of 30 minutes (1800 seconds), that means 100 x 8 x 50 x 1800 = **72 million iterations!**

Fortunately, there is a better way. When covering queues (and priority queues) in my Data Structures course this past spring, we touched on the idea of an event-driven simulation. **Epiphany: I didn't need to care about every second in a simulation, only those seconds where a call arrived or ended!**

The effect is that instead of 72 million iterations, I only needed 80 thousand (100 x 8 x 50 x 2). Much more manageable!

Armed with this knowledge, I did just that. I created an ``Event`` class, where ``Event`` represented either a call arriving or departing. I was cruising along and everything was going to be awesome.

**Then everything broke:**


.. figure:: /images/writing/lessons-learned-from-making-call-center-calculator/img-1.jpg


I searched all over. I had been refactoring, did my ``Event`` class definition exist in two separate files by chance? No. StackOverflow was a dead end. I was lost.

It took me a couple nights, probably 4-6 hours of debugging before the real problem finally dawned on me. It wasn't that I had defined my ``Event`` class twice, it was that my custom ``Event`` class was trying to redefine the DOM's ``Event`` interface!

https://developer.mozilla.org/en-US/docs/Web/API/Event

*Well, I felt like an idiot.*

The good news was that all I had to do was rename my class to resolve the problem. But lesson learned: the hardest errors almost always involve some oversight or misassumption on the programmer's part. 


2. Web Workers
==============
JavaScript is single threaded. (I learned) that when a script is running, no other scripts can run and the webpage appears frozen while any computation is done. Calculating Erlang C and computer simulated results can take 5-10 seconds, during which the website may appear frozen or stuck.

To provide a bit of feedback to the user, I thought to add a progress bar to the web app. However, how do I "update" it if JavaScript is single threaded?

**The answer is Web Workers.** A Web Worker is an object that runs a JavaScript file in a background thread. This way, the thread can continue doing all calculations without affecting the user interface. Futhermore, the Web Worker can pass messages to and from the code in which it is created, which seemed like a perfect way to bring my progress bar idea to life.

I'm no expert, but here's what I ended up with:

.. code-block:: JavaScript
    :linenos:

    // On Submit Functions - runs whenever form is submitted
    
    // Initialize worker and listener for form submission
    var worker = null;
    if (window.Worker) {
        worker = new Worker("/js/worker.js");
        document.getElementById('input_form').addEventListener('submit', startWorker);
    } else {
        alert("Workers are not supported. Please use a different browser.");
    }

    // Start worker
    function startWorker() {
        worker.postMessage(getInputs());
    }

    // Read form inputs into dictionary to pass to worker
    function getInputs() {
        let dict = {
            <get values from webform>
        };
        return dict;
    }

This is from my main JavaScript file, which contains all eventListeners. Any time the form is submitted, it creates a Web Worker and passes it a dictionary containing all the form's input values.

The Worker itself is set up to receive this dict, then it triggers the underlying Erlang C and Simulation code in the calc() function. Periodically, throughout the calc() function it passes back a message containing its overall progress. The Simulation portion has much greater time complexity than Erlang C, so for simplicity sake I calculate progress as the % of the simulation.

.. code-block:: JavaScript
    :linenos:

    self.addEventListener("message", onMessageReceive);

    function onMessageReceive(e) {
        let dict = e.data;
        calc(dict);
    }

    function calc(dict) {

        <calculate Erlang C>

       <calculate simulation> {
            // Update progress bar as you go
            var cur_progress = (Number(index) + 1) / x.length * 100;
            postMessage({type: "progress", value: cur_progress});
        }

        // Bundle everything up to pass back to main
        let results = {
            <dict containing Erlang C and simulation results>
        }
        postMessage({type: "complete", value: results});
    }

Finally, the main.js file has a simple switch function to take these progress updates and update the progress bar accordingly:

.. code-block:: JavaScript
    :linenos:

    // On Message Receipt Functions - runs whenever worker sends progress update or completes

    // Listener for when worker sends a message back
    worker.addEventListener("message", onMessageReceive);

    function onMessageReceive(e) {
        switch (e.data.type) {
            case "start":
                break;
            case "progress":
                updateProgress(e.data.value);
                break;
            case "complete":
                updateProgress(100);
                updateChart('chart', e.data.value);
                break;
            case "debug":
                console.log(e.data);
                break;
        }
    }

`Read more about Web Workers here`_. Pretty cool stuff!


3. JS import & importScript
===========================
Full disclosure:  originally I wrote all my JavaScript as one big file. I knew it was messy and figured I'd go back and clean things up later. Turns out, refactoring JavaScript is different than other languages like Python (where I was familiar with the idea of ``import package as pkg`` commands).

I'll save the narrative, but I learned along the way that the way JavaScript files are imported is less straightforward, especially when dealing with Web Workers. The tl;dr is:

    * ``import`` statements can only appear on root files
    * worker files, in part because they are not root files, have to rely on the earlier ``importScripts`` syntax.
    * To use ``import`` statements, you have to include ``type="module"`` in the html tag.

After some trial and error, I ended up breaking my JavaScript into the following five files:


    * **main.js** - top-level and called by the HTML. It contains all DOM event listeners and primarily serves as a controller between the page and underlying JS.
    * **chart.js** - code to initialize and update the Plotly chart
    * **worker.js** - code to receive and send messages to the Web Worker in main, as well as run the main calculations
    * **erlangc.js** - functions for calculating staffing according to Erlang C methodology
    * **simulation.js** - functions for calculating staffing according to a Simulation methodology

Is it good? I don't know. But it's an early programmer's attempt at staying modular and it works, so I'll give myself a passing grade.



.. _`Read more about Web Workers here`: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers