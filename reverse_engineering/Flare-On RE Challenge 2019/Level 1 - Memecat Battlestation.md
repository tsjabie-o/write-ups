After extracting the `.7z` file we are provided with two files: `MemeCatBattlestation.exe` and `Message.txt`. Let's take a look at the text-file first. In it, we find this text-fragment:

```
This is a simple game. Reverse engineer it to figure out what "weapon codes" you need to enter to defeat each of the two enemies and the victory screen will reveal the flag.
```

So off the bat we know that the `.exe` file will be some sort of game for which we'll have to find a set of two "weapon codes". Let's execute file and see what happens.

<img width="1175" alt="Pasted image 20250304160328" src="https://github.com/user-attachments/assets/e5f91699-e8b3-4c64-8499-e36dfe14a017" />

## Stage 1

The game is indeed very simple, with just a text-field and a button to interact with. When provided with a randomly chosen "weapon code", pressing the "Fire!"-button causes a text to pop up, saying "invalid weapon code". Since that's all the interaction we can perform, let's stop the executable and pick it apart.

When working with `.exe` or `.dll` files, I always start any analysis by taking inspecting the PE-headers. PE-headers contain useful metadata about the executable that could help us with more advanced analysis. I personally use PE-Bear to do this. 

An important piece of information that the PE-headers provide us with is the existence of the `.NET Hdr`, or .NEt-Header. This tells us that this executable uses the .NET runtime and as such has probably been programmed using `C#`. As such we'll be using a .NET-specific debugger / disassembler when reverse engineering.

First though, I always like to take a brief look through the strings contained in the executable. In this case, strings are just chains of bytes that fall within either the ASCII or Unicode range. Especially in this case, as we're looking for a "weapon code" which might very well be saved unencrypted as a string. PE-Bear automatically extracts all string from the binary.

Binaries contain many strings, but most of them are baked in automatically and are not saved as a result of the programmer explicitly creating them. In this case, I'll quickly scan through all of them to see if anything sticks out. We can always perform a more thorough guided search if needed later. It might take some experience to filter out built-in strings from programmer-created ones, but especially in a binary of a game about memes, cats and weapons, it might be a bit easier. It can also help to know that - in general - strings from the same origin live close together in a binary. Once you find strings that are clearly programmer-created, you know you're in (on of the) right region(s). 

Anyway, quite at the start of the binary you'll find strings like `explosionPictureBox`, `VictoryForm` and `Memecat Battlestation [Shareware Demo] - You parents credit card can unlock finul boss level please buy`, which are quite obviously custom. Among them, you'll also find a string `RAINBOW`, which catches the eye, as it's not a valid function or variable name and also does not have any obvious purpose.

<img width="1581" alt="Pasted image 20250304163046" src="https://github.com/user-attachments/assets/bd347b8d-c9ec-49e7-91b2-3558038d5d5c" />


Turns out this is indeed the weapon code we were looking for. Entering it and pressing the fire-button causes an animation to play in which an enemy cat is brutally killed by a rainbow-laser :(

However, this feels a bit like cheating. For learning's sake, let's also use a disassembler on the binary and see if we can find this `RAINBOW` that way as well. I'll be using `dnSpy` which is a disassembler and debugger for .NET. programs. When provided with a .NET binary, it will disassemble it and basically show us its source code, neatly organised into namespaces, methods, properties etc.

In the Assembly View we'll see a bunch of modules loaded. Each of these modules contain code that the binary uses. One of them is the code of the executable itself. Expanding that module, we see a few different namespaces.

<img width="442" alt="Pasted image 20250304203547" src="https://github.com/user-attachments/assets/082939d7-f865-4904-a320-ab890d53d65d" />


The namespaces `Stage1Form`, `Stage2Form` and `VictoryForm` seem important and their relation to the game is immediately quite clear. Since we're still focused on the first stage, let's pop open Stage 1.

<img width="391" alt="Pasted image 20250304203749" src="https://github.com/user-attachments/assets/6e04814b-e15f-4494-b5ad-51543aa6f571" />


We're met with a bunch of different functions, most of which have quite obvious functions. We can assume the weapon code we've entered will be verified at the moment we press the fire-button, so a logical place to start is the `FireButton_Click()` function. Immediately our smart thinking is rewarded as we find the following piece of code:

```CS
if (this.codeTextBox.Text == "RAINBOW"){
	...
	this.victoryAnimationTimer.Start();
	return;
}
```

## Stage 2
After completing the first stage we're met with yet another enemy cat in our territory and are asked to provide another weapon code. 

Let's immediately dive into `dnSpy` again and look under the `Stage2Form` namespace. If we navigate to the same `FireButton_Click()` function, we see that this time the `if`-statement has been replaced with the following:

```CS
if (this.isValidWeaponCode(this.codeTextBox.Text)){
	[..]
	this.victoryAnimationTimer.Start();
	return;
}
```

Looks like we need to put in some extra work this time. Let's open up this new `isValidWeaponCode()` function and see what it does.

```CS
private bool isValidWeaponCode(string s)
{
	char[] array = s.ToCharArray();
	int length = s.Length;
	for (int i = 0; i < length; i++)
	{
		char[] array2 = array;
		int num = i;
		array2[num] ^= 'A';
	}
	return array.SequenceEqual(new char[]
	{
		'\u0003',
		' ',
		'&',
		'$',
		'-',
		'\u001e',
		'\u0002',
		' ',
		'/',
		'/',
		'.',
		'/'
	});
}
```

Interesting, let's unpack what's happening here. On a higher level, we can see that the string we supply as a weapon code is compared to some other character array and the result is returned. Since we want this function to return `true` in order to win this next stage, we want this comparison to work out. Something happens in before the comparison is made though, let's look at all the details.

```CS
char[] array = s.ToCharArray();
int length = s.Length;
```

Here our string is turned into an array of characters, called `array`. Also its length is saved in the variable `length`. Simple.

```CS
for (int i = 0; i < length; i++)
{
	char[] array2 = array;
	int num = i;
	array2[num] ^= 'A';
}
```

Here, it iterates over the length of the array. In each iteration, enumerated by `i`, it creates a new character array called `array2` and sets it equal to `array`. Finally, at `array2[i]`, it sets the value to the result of an XOR operation with the character `A`.

What's important to know here is that under the hood, the variable `array` does not directly contain the character array, but rather a pointer to it. That means that by setting `array2 = array`, they both point to the same character array in memory. Thus when we set `array2[i]` to some value, this is equivalent to setting `array[i]` to that value. It seems like the entire existence of `array2` is meant to confuse someone analysing this code.

```CS
return array.SequenceEqual(new char[]
{
	'\u0003',
	' ',
	'&',
	'$',
	'-',
	'\u001e',
	'\u0002',
	' ',
	'/',
	'/',
	'.',
	'/'
});
```

Finally, we compare `array` (or `array2`) to this new character array with quite random characters. 

So now we know what happens: Our weapon code is XORed with the character 'A' and then is compared to this random character array. Since the XOR operation is transitive, the easiest way to find the correct weapon code is to XOR the random character array with the letter 'A'. 

Personally, I copied and slightly changed the supplied code to do this, but there's more tools out there. Here's the code I used:

```CS
public static void Main()
{
	char[] encr_array = {
		'\u0003',
		' ',
		'&',
		'$',
		'-',
		'\u001e',
		'\u0002',
		' ',
		'/',
		'/',
		'.',
		'/'
	};
	
	int length = encr_array.Length;
	
	for (int i = 0; i < length; i++)
	{
		char[] array2 = encr_array;
		int num = i;
		array2[num] ^= 'A';
	}
	Console.WriteLine(encr_array);
}
```

We find that the correct weapon code is `Bagel_Cannon`. Fill it in and we get to the victory screen!

Flag: `Kitteh_save_galixy@flare-on[.]com`
