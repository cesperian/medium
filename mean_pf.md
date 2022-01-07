Adding a Facebook login to your Angular app may seem passably easy; the tools available are relatively easy to use and complimented by comprehensive, well-presented documentation. In addition, Passport has a specific strategy for this task that can be found not only in their listing of strategies but also prominently mentioned in their own documentation.
The issue when developing using Angular (or any stack that not using sessions) is that these resources are mostly for a stateful endpoint. Strictly speaking, you could use the passport-facebook strategy for stateless authentication but it will add confusing, unnecessary code to the result. This is the proverbial ‘round-peg-in-square-hole’.

Disclaimer; While acknowledging that this article is lengthy for something that could be accomplished rather quickly, and that condensing it down to a series of steps ‘go here and press that’ would be preferable to some, after many years of teaching this subject to many people I also acknowledge a useful utility in understanding the logic behind steps taken, steps considered and steps avoided. It leads to an empowered, broader familiarity with a topic.

tldr; You’re gonna need a more patient temperament than that if you want to be a developer. Perhaps professional flaneur is a better suited calling.

0: Preface
You will not need much experience to get this functionality up and running. You should at least know what MEAN…well, means, and have a little history setting up a basic server using Express and some minimal Angular app that can exchange data with it.
If you are just starting out I would strongly recommend 2 courses by the exceptional, prolific Maximilian Schwarzmüller; Angular — The Complete Guide and NodeJS — The Complete Guide. The first is the definitive course on Angular, and the second is one the most comprehensive course available on the topic.
In case you are unfamiliar with Passport, it is authentication middleware for Express that has a extensive list of ‘strategies’ (532 at the time of this writing) that you can add to your app, so you only employ what’s needed. It’s extremely easy to use and can abstract away sometimes fairly complicated steps.

1. Open a Facebook developer account
Visit Facebook’s (Meta) developer site and open a new account if you don’t already have one. Under Settings => Basic are 4 properties that you need to address;
App ID and AppSecret; these are the two datapoints that allow your users to log into your app and for you to authenticate them, respectively. Treat the secret as you would any password.
Privacy Policy URL; This and User Data Deletion are 2 required fields. freeprivacypolicy.com is advised as one good, free service to generate a privacy policy but there is no shortage of them. User Data Deletion provides a way for Facebook users to request that their data be removed. This could be a simple url that leads to instructions on how this can be done, or a Data Deletion Request Callback.
Under Settings => Advanced => Upgrade API Version, make note of which API version your app is set to make requests to. Under Settings => Advanced => Security turn on Require App Secret. While this is not required in order to get your login working, having it enabled adds another safeguard that will be explained in step 3 when you add code that takes advantage of it.
Lastly, under Facebook Login => Settings => Client OAuth Settings => Allowed Domains for the JavaScript SDK is where you need to specify all the domains from which login requests will be originating from. So for your production site you would add https://www.yoursite.com, and for your local Angular server you would add localhost + port number; https://localhost:4200.
Observant readers will notice something potentially concerning; the protocol Facebook insists on using is https. Although this can still be disabled in the settings, as of 2018 Facebook requires that all apps use it. Good news traditionally first; hosting services like AWS and (the awesome) Heroku make this very easy, and enabling ssl for Angular’s local dev server could not be any simpler (explained in step 2). The bad news; once you compile your app and want to serve it up locally using your own Node server, now it will also need to use https since its localhost domain will be the one issuing login requests when serving up the compiled, static app files from the public directory. This can be a walk in the park for those who already know how to generate encrypted keys from the terminal without having to look it up, and a trip down the rabbit hole from hell for people currently Googling ‘ssl’. I am not enough of an authority to comment on how to best implement ssl, but thankfully Facebook has a whole page devoted to just this topic, and even endorses a service for this, Let’s Encrypt. In any case this speed bump only affects this intermediate step of your development where you may want to run integration testing.

2. Modify your Angular app and add a call to the sdk
This is where documentation can get confusing when making sense of the OAuth authentication flow in a stateless SPA scenario. Surprisingly, even the usually-comprehensive Facebook documentation comes up short when the Angular integration example stunningly uses v1 from 11 years ago. If I were a cynical man, I would mention that this may have something to do with Facebook being the company behind Angular’s competing technology, React. But i’m not so I won’t mention that.
The topic of OAuth itself could be a course on its own (and is). A good article on DigitalOcean’s site covers it in extracting but understandable detail.
To get started, you will need to serve your Angular app using https. Go to your app’s package.json file => scripts and add;

package.json
Start Angular’s dev server with npm run start:ssl.
The quickstart example in Facebook’s dashboard on how to add Facebook’s SDK to your app looks like this;

Which is all well and good until you realize there’s a very limited number of places you can place this. You’re not going to put it in a component template and although you could put it in the html doc to which your entire app is injected, adding anything to this root doc should be done exceedingly sparingly and for very good reason. Since “I didn’t know where else to put it” is a defense that can only range in lousiness depending on the specific situation, consider a better way; using Angular’s APP_INITIALIZER dependency-injection token.
From the Angular team’s own documentation;
The provided functions are injected at application startup and executed during app initialization. If any of these functions returns a Promise or an Observable, initialization does not complete until the Promise is resolved or the Observable is completed.
This is what you want to use as a storehouse for code that should run first as your app bootstraps. You don’t need to understand Observables or even JS Promises to take advantage of it.
Bonus tip; if you are new to Observables and Promises, you will eventually need to build a working understanding of them. The best course on this topic is RxJs In Practice by the wonderful Vasco Cavalheiro of Angular University.

2.1: Create a new TypeScript file, name it app.initializer.ts, and add the following;

app.initializer.ts. Most code courtesy of Facebook. Error markup courtesy of TSLint.
There is a lot of similarity between what is in here and what Facebook offers in its quickstart, and in fact you would copy paste code from their example and just add the extra trimmings as shown. However if we left it at this there would be a problem because ‘FB’ is currently an unrecognized variable.
At this point there are 2 things you could do, ranging from good to very good;
good; Add declare let FB: any outside the exported function ‘appInitializer’. This is telling the compiler, ‘trust me, variable FB exists’. It’s a convenient way to circumvent this issue. More information than you require can be found at TypeScript’s own documentation on it
very good; Install the @types/facebook-js-sdk package. In your terminal;
npm i -D @types/facebook-js-sdk
Add this type to the types array in your tsconfig.app.json file, and then restart the Angular server. Not only will it address the compilation issue, but your IDE can also now prompt you with FacebookStatic methods/properties wherever you invoke ‘FB’ in your project;

tsconfig.app.json
2.2: Add the token to the providers array of your app.module.ts and specify the exported initializer function;

app.module.ts
Bonus tip; At this point you can test your user’s Facebook login status in the initializer and specify a service to pass that result to. If you want to do this, you will need to add the service to the ‘deps’ array when providing the token in your module (thankfully, it is a lot less complicated than it may sound);

app.initializer.ts, with a service invoked

app.module.ts

2.3: Add the Facebook login functionality to your login component. Keep in mind that this will log a user in Facebook, allowing the app access to some personal data (eg; name, email). This doesn’t log the user into your app (that happens in step 3 when the token Facebook provides is sent to your own api, which authenticates it and then logs the user in or creates a new user account).
In plain English, it looks like this;

login.component.ts
Checking current login status can be done using FB.getloginStatus(). If you provide a function as an argument, that function will be given the status and then get called (this is wonderfully known as a ‘callback’);

By line 113, the value of status will be structured like this;

If status is connected as above, you can send the value of accessToken to your server, authenticate it with Facebook and then proceed to create a new user account or log into an corresponding existing user account. However, if the status is something else or the token does not exist, FB.login() needs to be called. This also takes a callback function that gets executed when complete;

There are a couple key things to notice here. The first is that not only is a callback function to FB.login being provided, but also a second argument object that allows you to specify options like ‘scope’. This is not required. If you omit it, the end result is that your server will only be provided two data points on the Facebook user profile; ‘name’ and ‘id’. Specifying scope is a way to request certain data, like an email address (note that email and public_profile are the only ones Facebook will freely provide to your app. If you want additional permissions, your app will need to be reviewed by Facebook).
Next, note that that the overall number of nested blocks is growing, and it doesn’t even include the full browser request logic yet. This is known as ‘callback hell’, and can be mitigated using Promises and/or Observables.

In other words, this could happen. Don’t let this happen.
For those not comfortable with these topics yet, you can just add the post requests and move on to step 3. However, for future reference remember that examples on refactoring a function with nested callbacks into one with Promises/Observables can be found below, before the final step 3.
Lastly, there are a number of ways to issue a post request to your server, and it’s largely preferential. The 3 most applicable options would be the simple Fetch API, a popular 3rd party package like Axios, or the venerable HttpClient available in Angular’s http module. Using the last option, the first request block becomes this;

The interesting Jason Watmore has many easy to understand examples of issuing requests using various methods across various platforms, his blog entry here uses Fetch and POST, with links to the others.
Bonus tip; complete code, with callback hell refactored out:

Using Promises and a little destructuring. Note that there is now only 1 code block for issuing a request

Using Observables with Promises and a bit more destructuring.

3. Add route controller logic and Passport strategy to your server
In plain English, the controller for the Express route your app targets looks like this;

3.1: The first part is simple, since the post data is contained in the body of the request object;

The second part requires a little more detail. We could skip it and proceed to send the accessToken for Facebook to authenticate, and there would be nothing wrong with that. However, the accessToken is a simple credential that can be compromised. From Facebook’s security documentation;
Almost every Graph API call requires an access token. Malicious developers can steal access tokens and use them to send spam from your app. Facebook has automated systems to detect this, but you can help us
If you remember the instruction back in step 1 to enable ‘Require App Secret’, this is why it was toggled on; to prevent anyone who has maliciously gained access to the token from being able to do anything with it.

Facebook dashboard > Settings > Advanced > Security
Create a Hmac object from your secret and token using NodeJS’ crypto package;

Bonus tip: it should probably go without saying, but if you are using version control with remote storage (eg; Git and GitHub) then you’re not going to want include sensitive data like secrets or database keys in your repository. Thankfully, the process to contain this data in a separate environment variables file, excluded from version control, is extremely easy thanks to the dotenv package.

3.2: Complete the code block by adding the following;

On line 123 we are specifying the data points we’re interested in. This is different from your Angular app including ‘scope’ when logging into Facebook, which was to notify the user what data points they are providing permission for our app to access.
Line 126 specifies which version of Facebook’s api you are using. You saw this in step 1 by going to Settings => Advanced => API Version in Facebook’s developer dashboard.
Line 124 through 131 may seem excessive, since the same result could be accomplished by merging all those lines into one, or indeed even into Axios’ get method. Of course, the result would be pretty ugly;

yuck.

Code style guides recommend, among many other things, limiting line length. The Google JavaScript Style Guide dictates capping the length at 80 characters, while TSLint will get annoyed after 140. Not only are shorter statements easier to read, they are easier to modify and debug.
Note that lines 128, 129 and 130 are contained in backtics, not single quotes, which allows the variable values to extrapolate to the string.
And that’s it! At line 133, fbResponse payload will contain a data property with the coveted data points like id, email, first_name, etc. You can use the id to register a new user or log an existing user into your app.
Except that it’s not quite it, not for an article billed as MEAN+P. It’s about 20% short.
There is no current implementation of a Passport strategy and at this point there is no clear reason to include one, even if there was a strategy appropriate for us to implement in this project, which there currently isn’t. So you certainly could call it a day with this feature and move onto your next project detail.
However, if you are allowing your users different methods to signup/login, and those methods are handled (or going to be handled) using Passport strategies, you will probably want to have your Facebook login logic contained in the same file as the rest of your auth logic. Of course, this isn’t required and also amounts to extra effort, but the result is that you won’t have some auth logic existing over here in one part of some controller and other auth logic somewhere else. So use Passport to compartmentalize your project appropriately…

3.3: ‘Employ a non-existent Passport strategy’. As established, the only Facebook-specific strategy available to Passport is ‘facebook-passport’. And while it may be terrific for some OAuth 2 implementation, it’s not applicable in this situation.
You could build and distribute your own strategy using the passport-strategy module to subclass the abstract Strategy class, but (mercifully) this is not necessary. One of the available strategies is passport-custom, which graciously enables you to authenticate using whatever custom logic you want.
Install both packages on your server; npm i -S passport passport-custom
In your main server file, require passport and then initialize it;

Between these two lines is where we sandwich whatever strategies needed, which when called upon would attach authenticated user data to the request object. However, you want to compartmentalize your auth logic not in one area of your server doc, but in its own standalone file. So create a new file, ‘auth.js’ and require it as well;

Scaffold auth.js like so;

auth.js
Any passport strategy your app uses can be contained in this single, exported module function. Pretty tidy. So far there is only one, ‘facebook-sdk-strategy’. The name is completely arbitrary, you may name this however you want. The real meat is in the second argument; an entirely-new, completely custom strategy created using the CustomStrategy class.
To complete the creation of this strategy, you provide a function as an argument to the new CustomStrategy. This function will get passed two objects; the original request and a callback (called ‘done’ in the below example) you will invoke when your logic is complete. The remainder of the code should look very similar, since it’s nearly the exact same as what was placed in your controller;

This custom strategy will extract and process the accessToken for any route into which it is placed, and return to that route the same request but with a user property added (assuming Facebook successfully authenticated the user). If the authentication fails Passport will automatically send a response with status 500;

auth.routes.js
The last thing to do is refactor the controller, removing most of the logic added previously because it is no longer needed; Passport uses our custom strategy to go through the motions of authenticating the accessToken and retrieving the data points before the controller is even reached. If the authentication runs into an error, the response is handled and the controller method is never invoked. So the method only needs to retrieve the Facebook data points from the user object;

Profit

Conclusion

And there it is. MEAN+P for Facebook logins in 3 steps. Easier than collecting underpants and vastly more profitable. Probably.
For readers who consider parts of this information incorrect or may not be explained in the best way, instead of leaving excoriating comments consider visiting the public repo I am using for articles, make suggested changes directly to the source and I will update the article with your insight.
Credit to Jason Watmore’s comprehensive Angular + Facebook example on StackBlitz.
