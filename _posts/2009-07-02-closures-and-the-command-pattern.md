---
layout: post
title: "Closures and the Command Pattern"
categories: c-sharp code design
---
Like a lot of OOP programmers, I'm a fan of [Design Patterns](http://www.c2.com/cgi/wiki?DesignPatterns). While I
don't treat it like the sacred tome that many think it is, I learned a lot of
design tricks for solving problems from it. As I've moved from C++ towards
other languages, it's become clear that many patterns in it exist just to get
around limitations in C++.

The anti-C++ crowd just uses this as evidence that C++ sucks since you have to
write a whole book to get around its shortcomings. Let's not go there.

Instead, I want to see if I can show you a bit about a language feature that
C++ lacks and that you may not know ([closures](http://en.wikipedia.org/wiki/Closure_%28computer_science%29)) by getting there from
something familiar to an average C++ OOP programmer (the [Command
Pattern](http://en.wikipedia.org/wiki/Command_pattern)) and changing it in stages. I'll use C# as the example language
here, but any language with closures would work: Lua, Scheme, etc.

## The Command Pattern

Let's say you're writing a chess program. You want it to support both regular
human players and built-in AI players. The core chess engine doesn't care
about what kind of players it's dealing wth. All it needs to know is for a
given player, what that player's move is.

For a human player, the UI code will get the user's move selection and pass
that to the engine. For an AI player, the AI rules select the best move and
pass it in.

The "move" here is the command pattern. The basic idea is that you have a
*command* class that encapsulates some procedure to perform and any data that
procedure needs. You can think of it like a function call and its parameters
bottled up together to be opened up later by someone else. A vanilla implementation is something like:

{% highlight csharp %}
interface ICommand
{
    void Invoke();
}

class MovePieceCommand : ICommand
{
    public Piece Piece;
    public int   X;
    public int   Y;

    public void Invoke()
    {
        Piece.MoveTo(X, Y);
    }
}
{% endhighlight %}

We'll also create a little command factory for creating the commands. This
isn't necessary now, but it'll make sense later when we start moving things
around.

{% highlight csharp %}
static class Commands
{
    // create a command to move a piece
    public static ICommand MovePiece(Piece piece, int x, int y)
    {
        var command = new MovePieceCommand();
        command.Piece = piece;
        command.X     = x;
        command.Y     = y;

        return command;
    }
}
{% endhighlight %}

To complete this first pass, here's a little block to test our code:

{% highlight csharp %}
class Program
{
    public static Main(string[] args)
    {
        // ui or ai creates command
        var piece = new Piece();
        var command =  Commands.MovePiece(piece, 3, 4);

        // chess engine invokes it
        command.Invoke();
    }
}
{% endhighlight %}

## The First Change: Delegates

The first change we'll make is a pretty minor one. The `ICommand` interface
only has a single method, `Invoke()`, so there's no real reason to make an
interface for it. Since delegates in C# work fine on instance methods, we can
just use that instead. We'll define a delegate type for a function that takes
no arguments and returns nothing, just like the `Invoke()` method in
`ICommand`.

{% highlight csharp %}
delegate CommandDel();

class MovePieceCommand
{
    public Piece Piece;
    public int   X;
    public int   Y;

    public void Invoke()
    {
        Piece.MoveTo(X, Y);
    }
}

static class Commands
{
    // create a command to move a piece
    public static CommandDel MovePiece(Piece piece, int x, int y)
    {
        var command = new MovePieceCommand();
        command.Piece = piece;
        command.X     = x;
        command.Y     = y;

        return command.Invoke;
    }
}

class Program
{
    public static Main(string[] args)
    {
        // ui or ai creates command
        var piece = new Piece();
        var command =  Commands.MovePiece(piece, 3, 4);

        // chess engine invokes it
        command();
    }
}
{% endhighlight %}

Not much different, although we did get to ditch the interface without any
loss in functionality.

## The Second Change: Ditch Invoke

That `Invoke()` method up there really doesn't do much. It just calls another
function. Let's see if we can pull that out. C# has "anonymous delegates",
which are basically functions defined within the body of another function.
We'll try that.

{% highlight csharp %}
class MovePieceCommand
{
    public Piece Piece;
    public int   X;
    public int   Y;
}

static class Commands
{
    // create a command to move a piece
    public static Action MovePiece(Piece piece, int x, int y)
    {
        var command = new MovePieceCommand();
        command.Piece = piece;
        command.X     = x;
        command.Y     = y;

        CommandDelegate invoke = delegate()
            {
                command.Piece.MoveTo(command.X, command.Y);
            };

        return invoke;
    }
}
{% endhighlight %}

Now instead of an `Invoke()` *method* we have an anonymous function stored in
a local `invoke` variable. But this local function isn't a method, so it
doesn't have a `this` reference. How does it get access to the
`MovePieceCommand` that stores the piece and location to move it to?

Like a little magic trick, the body of the `invoke` delegate actually accesses
the `MovePieceCommand` stored in `command`, a local variable defined *outside*
of itself in `MovePiece`. *That's* a closure: a local function that references
a variable defined outside of its scope.

In C#, the compiler will make sure those closed over local variables get moved
to the heap, so that our delegate still has access to them even after
`MovePiece` has returned. It actually does this by building a little class
like our old `MovePieceCommand` and turns the delegate we just made back into
a method. The advantage is that the *compiler* writes the class for us. We
don't have to. Call me lazy, but if there's anything I like, it's _doing
less_.

## Clean Up

By now it's clear `MovePieceCommand` isn't doing much. It's just a bag of
data. If our anonymous delegate can access local variables outside its scope
anyway, there's no reason to bundle them into an object. Let's kill it.

To clean things up a bit more, we'll also define the delegate using C#'s newer
[lambda syntax](http://msdn.microsoft.com/en-us/library/bb397687.aspx). It does the exact same thing, but more tersely.

{% highlight csharp %}
static class Commands
{
    // create a command to move a piece
    public static Action MovePiece(Piece piece, int x, int y)
    {
        return () => piece.MoveTo(x, y);
    }
}
{% endhighlight %}

And just like that, our whole command pattern has become a single line of
code.

## Conclusion

There are plenty of cases where it's still useful to implement a full command
pattern: maybe you need to be able to invoke the command in multiple ways, or
undo it. However, for simple problems, I go for the simple solution. If you're
working in a language with closures, it doesn't get simpler than this.
