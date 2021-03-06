{
   "thumbnail" : "/images/how-strong-is-your-password/thumb_password.jpg",
   "title" : "How strong is your password?",
   "date" : "2018-02-06T08:35:11",
   "authors" : [
      "david-farrell"
   ],
   "description" : "Password strength checking with zxcvbn",
   "draft" : false,
   "tags" : ["zxcvbn", "dropbox", "entropy", "data-password-zxcvbn"],
   "categories" : "security",
   "image" : "/images/how-strong-is-your-password/password.jpg"
}

In the latest [what-s new on CPAN]({{< relref "what-s-new-on-cpan---january-2018.md" >}}), I linked to [Data::Password::zxcvbn]({{<mcpan "Data::Password::zxcvbn" >}}), a new module which estimates the difficulty of cracking a given password. Developed by Gianni Ceccarelli, it's a port of Dropbox's JavaScript implementation by Dan Wheeler.

### How it works

Dan Wheeler's original blog [post](https://blogs.dropbox.com/tech/2012/04/zxcvbn-realistic-password-strength-estimation/) explains this in detail, but here's a quick summary. To estimate a password's strength we measure its [entropy](https://en.wikipedia.org/wiki/Password_strength#Entropy_as_a_measure_of_password_strength). The zxcvbn algorithm estimates a password's entropy in three stages:

1. match - matches parts of a password against patterns like: known words, sequences, dates etc
2. score - for every matched pattern, calculate its entropy
3. search - find the lowest entropy sequence of non-overlapping matches

Imagine the password "baseball2005-59xH}"; zxcvbn will match "baseball" in its popular words dictionary, so its entropy is quite low. Similarly, "2005" will match the year pattern. On the other hand, the last part "-59xH}" isn't in our word list, nor does it match any of zxcvbn's common patterns and would require brute-force guessing, which has a very high entropy. Thus zxcvbn's entropy for the password would be in pseudo code:

    dictionary_entropy("baseball") + year_entropy("2005") + brute_force_entropy("59xH{")

Once it has calculated the entropy of a password, zxcvbn estimates other statistics like how many guesses it would take to crack the password, and the password cracking time.

### Installation

Data::Password::zxcvbn is on CPAN, so installation can be done with your favorite CPAN client, like [cpan]({{<mcpan "App::CPAN" "cpan" >}}) or [cpanm]({{<mcpan "App::cpanminus" "cpanm" >}}).

    $ cpanm Data::Password::zxcvbn

### How to use it

Data::Password::zxcvbn is easy to use. It exports a function called `password_strength` which accepts a password string parameter. It then returns a hashref containing useful [information]({{<mcpan "Data::Password::zxcvbn#Return-value" >}}) about the strength of the password.

For example, this quick script accepts a password argument, and prints how strong it's estimated to be:

```perl
#!/usr/bin/env perl
use Data::Password::zxcvbn 'password_strength';

my $est_strength = password_strength(shift @ARGV);
printf "Password strength [0-4]: %d, # guesses needed: %d\n",
       $est_strength->{score},
       $est_strength->{guesses},
```

The `score` is a value between 0 and 4, indicating the estimated password strength, with 0 being ridiculously easy, and 4 being super strong. The `guesses` value is an estimated number of guesses needed to correctly guess the password. Keep in mind that depending on how the password is encrypted, and where it is stored, attackers might be able to make billions of guesses per second. So the `guesses` number needs to be really large to be considered secure. Saving the script as `password-strength`, I can run it as follows:

    $ ./password_strength foobar
    Password strength [0-4]: 0, # guesses needed: 915

Oh no! My password "foobar" was rated 0. I suppose it's not very strong. Let's try another favorite example password of mine:

    $ ./password_strength itsasecret
    Password strength [0-4]: 1, # guesses needed: 29445

Hmm "itsasecret" was rated a little stronger, but not by much. Let's try a UUID (generated by [Data::UUID]({{<mcpan "Data::UUID" >}})):

    $ ./password_strength 7E943948-0C75-11E8-A90E-9860F82DAED4
    Password strength [0-4]: 4, # guesses needed: -1

Ding ding ding, we have a winner. The UUID was given the highest score of 4, and the estimated number of guesses was infinite (-1). I'll never remember it though: I should probably use a password manager.


### Detecting fragile passwords

Some passwords appear strong, but contain data associated with its user, making them fragile. Like my employer's domain name:

    $ ./password_strength ziprecruiter.com
    Password strength [0-4]: 4, # guesses needed: 949660000000

So [ziprecruiter.com](https://ziprecruiter.com) is considered a strong password, but it's not a strong password for _me_. How can we detect these instances of fragile passwords?

Data::Password::zxcvbn's `password_strength` function accepts a hashref of options as a second parameter. One option is called "user_input", which can be used to supply data known about the user, such as their name, date of birth etc. I've updated the script to read this from the command line:

```perl
#!/usr/bin/env perl
use Data::Password::zxcvbn 'password_strength';

my $est_strength = password_strength(shift @ARGV, { user_input => { @ARGV }});
printf "Password strength [0-4]: %d, # guesses needed: %d\n",
       $est_strength->{score},
       $est_strength->{guesses};
```

Running the script and including my employer data, you can see the password's strength is now rated much lower:

    $ ./password_strength ziprecruiter.com employer ziprecruiter
    Password strength [0-4]: 2, # guesses needed: 1010000

If you were using Data::Password::zxcvbn in a web application as part of a user registration form (or a password reset feature), you could pass the user's details to `password_strength` to catch these instances of fragile passwords. For users with admin privileges, you might require that their zxcvbn password score is the highest (4). This seems more secure than developing your own password validation function to check a password is of a minimum length, with a certain combinations of characters.

### References

* CPAN: [Data::Password::zxcvbn]({{<mcpan "Data::Password::zxcvbn" >}}) and Bitbucket [repo](https://bitbucket.org/broadbean/p5-data-password-zxcvbn/)
* Original Dropbox blog [post](https://blogs.dropbox.com/tech/2012/04/zxcvbn-realistic-password-strength-estimation/) discussing zxcvbn
* The Usenix Symposium zxcvbn talk [page](https://www.usenix.org/conference/usenixsecurity16/technical-sessions/presentation/wheeler) with video and paper
* The zxcvbn [repo](https://github.com/dropbox/zxcvbn)

\
Cover image by [freepik](https://www.freepik.com/free-vector/red-lock-with-password_715015.htm)
