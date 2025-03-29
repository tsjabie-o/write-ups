## The challenge
This level of the FlareOn Challenge is about an Android application. It's a fairly simple challenge, but it's a great introduction to Android reverse engineering, which I myself was not that familiar with. We'll use [JADX](https://github.com/skylot/jadx) to view the source code of the `.apk` and the [Android Studio SDK](https://developer.android.com/studio) to emulate it.

The folder contains two files again: 
- `flarebear.apk`
- `Message.txt`

The text file contains the following clue: 
```
We at Flare have created our own Tamagotchi pet, the flarebear. He is very fussy. Keep him alive and happy and he will give you the flag.
```

## Emulating the .apk
The first thing we'll do is just emulate and run the `flarebear.apk` file. Again, we can use Android Studio to spin up a virtual Android environment and install the apk file. When first starting the application, we'll be met with the following screen. 

![Pasted image 20250329213021](https://github.com/user-attachments/assets/ccc92cd5-1dc9-45ef-9fdc-3591cd35bbeb)


Let's create a new flare bear! We'll call it `mybear`.

![New_Bear](https://github.com/user-attachments/assets/075f0e85-5ba4-4b2e-845c-3958d746ef3f)


We can press different buttons to perform different actions with the bear. The first one makes them eat, the second one makes them play with a ball and the third one cleans up any poop that has accumulated on screen, which seems to happen some time after feeding the bear. We can notice that the bears shows two different emotions, happy or sad.

Of course, the text file hinted at needing to keep the bear happy and alive. Playing around a bit with the different actions, it's not immediately obvious what makes the bear happy or sad. Furthermore, we've already seen the bear happy and alive and no flag has revealed itself yet.

## Analysis with JADX
It's time to dive into the source code and find out what's going on under the hood. I'll use the cli version of `jadx` to unpack the `apk` and use VS Code to look at the source code, but there's also the GUI option.

```bash
jadx flarebear.apk -d out
```

One of the first things to look at while reverse engineering an `.apk` file is the `AndroidManifest.xml`. This contains information about what sections of the codebase could be interesting. For now, look at the following snippet:

```xml
<activity android:name="com.fireeye.flarebear.FlareBearActivity"/>
<activity
	android:name="com.fireeye.flarebear.NewActivity"
	android:noHistory="true"/>
<activity android:name="com.fireeye.flarebear.CreditsActivity"/>
<activity android:name="com.fireeye.flarebear.MainActivity">
```

These so-called *activities* are core components of the application. They represent individual UI elements. Usually, you can view them as equivalent to multiple `main` functions in C code. The strings that are declared as `android:name` are the source files in which we can find the activities' code. Out of the four activities here, `FlareBearActivity` seems most promising to contain code relating to the logic of the bear's emotion. In the `source` directory that `jadx` made, we'll find the code.

We see a whole bunch of different functions. Some that stand out as important to the internal logic of the bear are:
- `feed()`
- `play()`
- `clean()`
- `setMood()`
- `isHappy()`
- `isEcstatic()`
- `danceWithFlag()`

These two functions also might be interesting to look at:
- `getPassword()`
- `decrypt()`

Let's start with the `danceWithFlag()` function, as that sounds like the final goal here. Looking at the code inside the function, it seems to call the `decrypt()` and `getPassword()` function twice as well as load some resources. At first glance, especially when also looking at the `getPassword()` function, it would be quite hard to figure out exactly what's happening here. So before doing that, let's see if we can find out how we get this function to be called in the first place. Considering this is only the third challenge, this is probably enough, but we can always return later.

There is only one function which calls `danceWithFlag()` and that is `setMood()`.

```java
public final void setMood() {
	if (isHappy()) {
	[..].setTag("happy");
		if (isEcstatic()) {
			danceWithFlag();
			return;
		}
		return;
	}
}
```

Alright, this immediately tells us our goal quite clearly: we need both the `isHappy()` and the `isEcstatic()` function to return `True`.

```java
public final boolean isHappy() {
	double stat = getStat('f') / getStat('p');
	return stat >= 2.0d && stat <= 2.5d;
}

public final boolean isEcstatic() {
	return getState("mass", 0) == 72 && getState("happy", 0) == 30 &&
	getState("clean", 0) == 0;
}
```

Let's start with `isHappy()`. This function returns true based on the results of `getStat('f')` and `getStat('p')`. 

```java
public final int getStat(char activity) {
	String act = [..].getString("activity", "");
	Intrinsics.checkExpressionValueIsNotNull(act, "act");
	String str = act;
	int i = 0;
	for (int i2 = 0; i2 < str.length(); i2++) {
		if (str.charAt(i2) == activity) {
			i++;
		}
	}
	return i;
}
```

It seems like this `getStat()` function counts how many time the argument character occurs in the `"activity"` string. Since we're seeing the letters `f` and `p`, we could guess that these relate to the feeding and playing buttons. If we then look at the `play()`, `feed()` and `clean()` functions, that guess is made even stronger.

```java
public final void feed(@NotNull View view) {
	[..]
	saveActivity("f");
	changeMass(10);
	changeHappy(2);
	changeClean(-1);
	[..]
}
public final void play(@NotNull View view) {
	[..]
	saveActivity("p");
	changeMass(-2);
	changeHappy(4);
	changeClean(-1);
	[..]
}
public final void clean(@NotNull View view) {
	[..]
	saveActivity("c");
	changeMass(0);
	changeHappy(-1);
	changeClean(6);
	[..]
}

```

For now I'd say it's safe to assume that the `"activity"` string is a list of characters resembling all of the actions we've taken so far. With that knowledge, we need the amount of times we feed the bear to be between 2 and 2.5 greater than the amount of times we play with the bear, in order to achieve happiness. 

We can quickly confirm this by creating a new bear and - for example - pressing the feed-button 10 times and the play-button five times and seeing the bear smile :)

Next, to make the bear ecstatic. We need the `getState("mass", 0)`, `getState("happy", 0)` and `getState("clean", 0)` to return specific values; 72, 30 and 0 respectively. Looking at the code in the `feed()`, `play()` and `clean()` functions again, it is safe to assume that `changeMass()`, `changeHappy()` and `changeClean();` functions change some internal state variable, which can be read by calling `getState()`. 

Now we know how to get `danceWithFlag()` to be called. We need to perform each action (`f, p, c`) a specific amount of times such that these constraints are met:
- `2 <= (f / p) <= 2.5`
- `mass == 72`
- `happy == 30`
- `clean == 0`

## Calculating the right amount of actions

There are multiple ways we could get around finding the right combination of actions. Personally, I immediately see the last three constraints as a system of linear equations that can be solved (explanation of what that means is beyond the scope of this write-up, but you could start [here](https://en.wikipedia.org/wiki/System_of_linear_equations)). We can use `numpy` in Python to do this for us.

```python
import numpy as np

A = np.array([
	[10, -2, 0],
	[2, 4, -1],
	[-1, -1, 6]
])
B = np.array(
	[72, 30 , 0]
)

solution = np.linalg.solve(A, B)
print(f"Feed: {round(solution[0])}, Play: {round(solution[1])}, Clean: {round(solution[2])}")
```

```
Feed: 8, Play: 4, Clean: 2
```

Great, this also satisfies our first constraint!

When we perform exactly these actions, we indeed see the flag:

![Flag](https://github.com/user-attachments/assets/afd7ffde-e5af-4024-8483-c3e8e8834ec0)
