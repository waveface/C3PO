# C3PO: Hassle-free Localization

##	Philosophy

Localization is hard. A good app localization process **must:**

*	…be fully automatic.
*	…reflect code changes automatically.  
*	…only use **one** set of XIB/NIB files.
*	…only use *one* `Localization.strings` file per locale.
*	…only use UTF-8.


C3PO does exactly that.  Under the hood, it does these things:

1.	It calls `genstrings` to create a table of strings to be used in `Localization.strings`
2.	It calls `ibtool` to extract all the strings from the XIBs
3.	It merges results from step 1 and 2.

##	Conventions to follow, and their *raison d’être*

*	**Use CamelCase or WIDE_CASE for `NSLocalizedString()` params.**

	A full sentence might be legible, but things change and identical sentences or words in one locale may look totally different in another.
	
	You can use either CamelCase or `WIDE_CASE`, but `WIDE_CASE` is recommended:
	
		- (id) initWithNibName:(NSString *)name bundle:(NSBundle *)bundle {
			
			self = [super initWithNibName:name bundle:bundle];
			if (!self)
				return nil;
			
			self.title = NSLocalizedString(@"USERS_LIST_TITLE", @"Title for users list");
		
		}

*	**Don’t localize your XIB files.**

	You’ll be in it with a bag of hurt.  Just never, never do that.
	
	Let me tell a story.
	
	*	You localized a particular XIB in English, French and German.
	
	*	You included a diligently maintained external library, which is localized in English, Japanese, and French.
	
	*	Since iOS frameworks are not available, the library does not provide its own bundle, and you’re good, faithful and generally in a hurry, you put the whole thing right into the project.
	
	Now you get angry letters from German and Japanese users.  The app don’t even run for them.
	
	Not convinced?  — how about that after fixing this issue (wasted two days that could be spent coding), you need a bunch of testers to go through every single locale you support, and all those you don’t but your users use, for *every release,* simply because *your UI is localized in the wrong way?*
	
	In the world of Xcode, **every single localized resource** for an app essentially vouches *the entire application* capable of working in a particular locale.  There is such thing as a fallback locale on iOS.
	
	When your user is running the app, and the app is capable of dealing with the user’s locale, the app runs under that locale.  In our horror story, the user is Japanese, so the app runs in Japanese.  If something is not available in the `ja_JP` locale (for example, your XIBs), it’s not available.  `-[UINib nibWithName:bundle:]`, for example, throws an exception on this.
	
*	**Localize only what you need.**

	It’s no fun testing your app for a language you probably don’t know very well.  And it’s expensive to bounce it back and forth between testers — both time and effort.
	
	Imagine this:
	
	>	“My app just broke.”  *(The user is not a native English speaker.)*
	>	
	>	“Okay, let’s enable debug mode and see what happens…”
	>	
	>	“It says ‘Fin del seguimiento de la pila de la excepción interna’…”
	
	Now you need to learn Spanish.


##	Using C3PO

In your Xcode project, create a new **Run Script** Build Phase.  Put it before the **Copy Files** build phase so your product contains fresh bits.  If you like `zsh`, use this:

	find ${SRCROOT} -name "Localizable.strings" -print | xargs -I {} echo "iconv -f UTF-8 -t UTF-16 =(cat {})> {}"  | zsh
	python ${SRCROOT}/externals/C3PO/localize.py # Wherever the C3PO script is
	find ${SRCROOT} -name "Localizable.strings" -print | xargs -I {} echo "iconv -f UTF-16 -t UTF-8 =(cat {})> {}"  | zsh

We wrap the line doing “real” work between conversions from and to UTF-16 for these reasons:

*	UTF-16 strings, although used widely by Apple tools (for example, `ibtool`), often show up garbled on external systems, and wreaks havoc with some Git clients, even on a Mac.

*	The translator may be using Windows.

So, the best thing to do is to use UTF-8.  We use `iconv` to convert it to UTF-16, before calling C3PO, and convert it back to UTF-8 so it looks good for external use.

If you use Bash, or if you simply *can’t* include `zsh` because you’re writing code *for other people,* something along this line will work:

	cd ${SRCROOT}

	find . -name "Localizable.strings" -not -path "*External*" -print | xargs -I {} echo "iconv -f UTF-8 -t UTF-16 {} > {}.new; mv -f {}.new {}" | sh -xv
	python ${SRCROOT}/External/C3PO/localize.py
	find . -name "Localizable.strings" -not -path "*External*" -print | xargs -I {} echo "iconv -f UTF-16 -t UTF-8 {} > {}.new; mv -f {}.new {}" | sh -xv

###	Having XIBs localized 

You can leave labels populated with strings like USER_INFO_LABEL_TITLE, and same with buttons.  Just make sure you use a custom subclass.  Then, override `-awakeFromNib`, and say something like

	self.text = NSLocalizedString(self.text, nil)

Or, if you like outlet collections and are not afraid of cobweb code:

	for (UILabel *aLabel in self.interfaceLabels)
		aLabel.text = NSLocalizedString(aLabel.text, @"C3PO Localized");

This can be dynamically injected into the class hierarchies in a later release, but it’s eight in the morning, I haven’t slept, and my train departs at 13:00, so we’ll leave this for the next release. 

Time to take your app to the world.

##	Credits

*	[@jamex](http://twitter.com/james) wrote the entire project.
*	[@evadne](http://twitter.com/evadne) forked this, revised README, added Bash stuff, and told a couple of stories.
