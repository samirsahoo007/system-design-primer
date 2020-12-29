When main-stream data structures like Lists, Maps, Sets, Trees etc. are mostly used for achieving certain results about whether the data exist or not, maybe along with their number of occurrences and such, Probabilistic data structures will give you memory-efficient, faster result with a cost of providing a ‘probable’ result instead of a ‘certain’ one. It might not seems intuitive to use such data structures for now, but I’ll try to convince you in this post that these type of data structures have their specific use cases and you might find them useful in certain scenarios.

# Bloom filter

Do you know how hash tables work? When you insert a new data in a simple array or list, the index, where this data would be inserted, is not determined from the value to be inserted. That means there is no direct relationship between the ‘key(index)’ and the ‘value(data)’. As a result, if you need to search for a value in the array you have to search in all of the indexes. Now, in hash tables, you determine the ‘key’ or ‘index’ by hashing the ‘value’. Then you put this value in that index in the list. That means the ‘key’ is determined from the ‘value’ and every time you need to check if the value exists in the list you just hash the value and search on that key. It’s pretty fast and will require O(1) searching time in Big-O notation. 

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash.png)

Now, let’s consider that you have a huge list of weak passwords and it is stored on some remote server. It’s not possible to load them at once in the memory/RAM because of the size. Each time a user enters his/her password you want to check if it is one of the weak passwords and if it is, you want to give him/her a warning to change it to something stronger. What can you do? As you already have the list of the weak passwords, you can store them in a hash table or something like that and each time you want to match, you can check against it if the given password has any match. The matching might be fast but the cost of searching it on the disk or over the network on a remote server would make it slow. Don’t forget that you would need to do it for every password given by every user. How can we reduce the cost?

Well, Bloom filter can help us here. How? I’m going to answer it after explaining how a bloom filter works. OK?

By definition, Bloom filter can check for if a value is ‘possibly in the set’ or ‘definitely not in the set’. The subtle difference between ‘possibly’ and ‘definitely not’ — is very crucial here. This ‘possibly in the set’ is exactly why it is called probabilistic. Using smart words it means that false positive is possible (there can be cases where it falsely thinks that the element is positive) but false negative is impossible. Don’t be impatient, we are explaining what does it actually mean, shortly.

The bloom filter essentially consists of a bit-vector or bit-list(a list containing only either 0 or 1-bit value) of length m, initially all values set to 0, as shown below.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash1.png)

To add an item to the bloom filter, we feed it to k different hash functions and set the bits to ‘1’ at the resulting positions. As you can see, in hash tables we would’ve used a single hash function and as a result get only a single index as output. But in the case of the bloom filter, we would use multiple hash functions, which would give us multiple indexes.


![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash2.png)

As you can see in the above example, for the given input ‘geeks’ our 3 hash functions will give 3 different output — 1, 4 and 7. We’ve marked them.


![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash3.png)

For another input ‘nerd’, the hash functions give us 3, 4 and 5. You might’ve noticed that the index ‘4’ is already marked by the previous ‘geeks’ input. Hold your thought, this point is interesting and we’re going to discuss it shortly.

We’ve already populated our bit vector with two inputs, now we can check for a value for its existence. How can we do that? 
Easy. Just as we would’ve done it in a hash table. We would hash the ‘searched input’ with our 3 hash functions and see what are the resulting indexes hold.


![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash4.png)

So, searching for ‘cat’, our hash functions are giving us 1, 3 and 7 this time. And we can see that all of the indexes are already marked as 1. That means we can say, “maybe ‘cat’ is already inserted on our list”. But it didn’t. So, what’s went wrong? 
Actually, nothing went wrong. The thing is, this is the case of a ‘false positive’. Bloom filter is telling us that it seems that maybe ‘cat’ was inserted before, because the indexes should’ve been marked by ‘cat’ are already marked (though by other different data).
So, if that’s the case, how it is helpful? Well, let’s consider if ‘cat’ would’ve given us the output of 1, 6, 7 instead of 1, 3, 7, what would happen then? We can see that among 3 indexes, 6 is ‘0’, that means it wasn’t marked by any of the previous inputs. That means obviously ‘cat’ never inserted before, if it was, there was no chance of 6 to be ‘0’, right? That’s how bloom filter can tell ‘certainly’ if a data is not on the list.

So, in a nutshell:

    If we search for a value and see any of the hashed indexes for this value is ‘0’ then, the value is definitely not on the list.
    If all of the hashed indexes is ‘1’ then ‘maybe’ the searched value is on the list.

Does it start making sense? A little maybe?

Fine, now, back to the ‘password’ example we were talking earlier. If we implement our weak password checking with this type of bloom filter, you can see that initially, we would mark our bloom filter with our list of passwords, which will give us a bit vector with some indexes marked as ‘1’ and others left as 0. As the size of the bloom filter won’t be very large and will be a fixed size, it can easily be stored in the memory and also on the client side if necessary. That’s why bloom filter is very space-efficient. Where a hash table requires being of arbitrary size based on the input data, the bloom filters can work well with a fixed size. 
So, every time a user enters their password, we will feed it to our hash functions and check it against our bit vector. If the password is strong enough, the bloom filter will show us that the password is certainly not in the ‘weak password list’ and we don’t have to do any more query. But if the password seems weak and gives us a ‘positive’ (might be false positive) result we will then send it to our server and check our actual list to confirm.

As you can see, most of the time we don’t even need to make a request to our server or read from disk to check the list, this will be a significant improvement in speed of the application. In case, if we don’t want to store the bit-vector at the client side, we can still load it in the server memory and that will at least saves some disk lookup time. Also consider that, if your bloom filters false positive rate is 1%(we will talk about the error rate in details later), that means among the costly round-trips to the server or the disk, only 1% of the query will be returned with false result, other 99% won’t go in vain. 


Not bad, huh?


![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash5.png)

Bloom filter operations

The basic bloom filter supports two operations: test and add.

Test is used to check whether a given element is in the set or not.

Add simply adds an element to the set.

Now a little quiz for you.

Based on what we’ve discussed so far, is it possible to Remove an item from the bloom filter? If yes, then how?

Take a 2 minutes break and think about the solution.

Got anything? Nothing? Let me help you a bit. Let’s bring back the bit-vector after inserting ‘geeks’ and ‘nerd’ in it.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash6.png)

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash7.png)

Now we want to remove ‘geeks’ from it. So, if we remove 1, 4, 7 from the bit vector, as they are marked by ‘geeks’, and convert them to ‘0’, what will happen? You can easily see that, next time if we search for ‘nerd’, as the index ‘4’ will show ‘0’, it will definitely tell us that ‘nerd’ is not on the list, though it actually is. That means removal is impossible without introducing false negatives.

So, what’s the solution?

The solution is we can’t support Remove operation in this simple bloom filters. But if we really need to have a Removal functionality we can use a variation of the bloom filter known as ‘Counting bloom filter’. The idea is simple. Instead of storing a single bit of values, we will store an integer value and our bit vector will then be an integer vector. This will increase the size and costs more space to gives us the Removal functionality. Instead of just marking a bit value to ‘1’ when inserting a value, we will increment the integer value by 1. To check if an element exists, check if the corresponding indexes after hashing the element is greater than 0. 
If you are having a hard time to understand how a ‘Counting bloom filter’ can give us ‘deletion’ feature, I’ll suggest you take a pen and a paper and simulate our bloom filter as a counting filter and then try a deletion on it. Hopefully, you’ll get it easily. If you failed, try again. If you failed again then please leave a comment and I’ll try to describe it.
Bloom filter size and number of Hash functions

You might already understand that if the size of the bloom filter is too small, soon enough all of the bit fields will turn into ‘1’ and then our bloom filter will return ‘false positive’ for every input. So, the size of the bloom filter is a very important decision to be made. A larger filter will have less false positives, and a smaller one more. So, we can tune our bloom filter to how much precise we need it to be based on the ‘false positive error rate’. 
Another important parameter is ‘how many hash functions we will use’. The more hash functions we use, the slower the bloom filter will be, and the quicker it fills up. If we have too few, however, we may suffer too many false positives.


![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash8.png)

You can see from the above graph that, increasing the number of hash functions, k, will drastically reduce the error rate, p.

We can calculate the false positive error rate, p, based on the size of the filter, m, the number of hash functions, k, and the number of elements inserted, n, with the formula:


![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash9.png)

Seems like WTF? Don’t worry, we would actually mostly need to decide what our m and k would be. So, if we set an error tolerance value p and the number of elements n by ourselves we can use the following formulas to calculate these parameters:


![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash10.png)

Another important point I also need to mention here. As the sole purpose of using bloom filter is to search faster, we can’t use slow hash functions, right? Cryptographic hash functions such as Sha-1, MD5 won’t be good choice for bloom filters as they are a bit slow. So, the better choices from the faster hash function implementations would be murmur, the fnv series of hashes, Jenkins hashes and HashMix.
Applications

Bloom filter is all about testing Membership in a set. The classic example of using bloom filters is to reduce expensive disk (or network) lookups for non-existent keys. As we can see that bloom filters can search for a key in O(k) constant time, where k is the number of hash functions, it will be very fast to test non-existence of a key.

If the element is not in the bloom filter, then we know for sure we don’t need to perform the expensive lookup. On the other hand, if it is in the bloom filter, we perform the lookup, and we can expect it to fail some proportion of the time (the false positive rate).

For some more concrete examples:

    You’ve seen in our given example that we could’ve use it to warn the user for weak passwords.
    You can use bloom filter to prevent your users from accessing malicious sites.
    Instead of making a query to an SQL database to check if a user with a certain email exists, you could first use a bloom filter for an inexpensive lookup check. If the email doesn’t exist, great! If it does exist, you might have to make an extra query to the database. You can do the same to search for if a ‘Username is already taken’.
    You can keep a bloom filter based on the IP address of the visitors to your website to check if a user to your website is a ‘returning user’ or a ‘new user’. Some false positive value for ‘returning user’ won’t hurt you, right?
    You can also make a Spell-checker by using bloom filter to track the dictionary words.
    Want to know how Medium used bloom filter to decide if a user already read post? Read this mind-blowing, freaking awesome article about it.


Ref: https://hackernoon.com/probabilistic-data-structures-bloom-filter-5374112a7832

