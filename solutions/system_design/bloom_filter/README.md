When main-stream data structures like Lists, Maps, Sets, Trees etc. are mostly used for achieving certain results about whether the data exist or not, maybe along with their number of occurrences and such, Probabilistic data structures will give you memory-efficient, faster result with a cost of providing a ‘probable’ result instead of a ‘certain’ one. It might not seems intuitive to use such data structures for now, but I’ll try to convince you in this post that these type of data structures have their specific use cases and you might find them useful in certain scenarios.

The large part of the reason behind Bloom filter popularity is that they have that combination of being a fairly simple data structure to design and implement that is also very useful in many contexts. They were invented in 1970s by Burton Bloom[1],[2] but they only really “bloomed” in the last few decades with the onslaught of large amounts of data in various domains, and the need to tame and compress such huge datasets.

One simple way to think about Bloom filters is that they support insert and lookup in the same way the hash tables do, but using very **little space**, i.e., one byte per item or less. **This is a significant saving when you have many items and each item takes up, say 8 bytes.**

# Use case I:

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom-filters_01.png)
Figure 1: Bloom filters in distributed storage systems.(See explanation below) 


# Use case II:

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom-filters_02.png)
Figure 2: Usage of Bloom filter in Squid web proxy. 


# Bloom filter

Bloom filters do not store the items themselves and they use less space than the lower theoretical limit required to store the data correctly, and therefore, they exhibit an error rate. **They have false positives but they do not have false negatives**, i.e. and the one-sidedness of this error can be turned to our benefit. When the Bloom filter reports the item as Found/Present, **there is a small chance that it is not telling the truth, but when it reports the item as Not Found/Not Present, we know it’s telling the truth**. So, **in the context where the query answer is expected to be Not Present, Bloom filters offer great accuracy most of the time plus space-saving benefits**.

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

## Bloom filter operations

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

**The solution is we *can't support Remove* operation in this simple bloom filters**. But if we really need to have a Removal functionality we can use a variation of the bloom filter known as *Counting bloom filter*. The idea is simple. Instead of storing a single bit of values, we will store an integer value and our bit vector will then be an integer vector. This will increase the size and costs more space to gives us the Removal functionality. Instead of just marking a bit value to ‘1’ when inserting a value, we will increment the integer value by 1. To check if an element exists, check if the corresponding indexes after hashing the element is greater than 0. 

### Bloom filter size and number of Hash functions

You might already understand that if the size of the bloom filter is too small, soon enough all of the bit fields will turn into ‘1’ and then our bloom filter will return ‘false positive’ for every input. So, the size of the bloom filter is a very important decision to be made. *A larger filter will have less false positives*, and a smaller one more. So, we can tune our bloom filter to how much precise we need it to be based on the ‘false positive error rate’. 

Another important parameter is ‘how many hash functions we will use’. The more hash functions we use, the slower the bloom filter will be, and the quicker it fills up. If we have too few, however, we may suffer too many false positives.


![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash8.png)

You can see from the above graph that, increasing the number of hash functions, k, will drastically reduce the error rate, p.

We can calculate the false positive error rate, p, based on the size of the filter, m, the number of hash functions, k, and the number of elements inserted, n, with the formula:


![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash9.png)

Seems like WTF? Don’t worry, we would actually mostly need to decide what our m and k would be. So, if we set an error tolerance value p and the number of elements n by ourselves we can use the following formulas to calculate these parameters:


![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/bloom_filter_hash10.png)

Another important point I also need to mention here. As the sole purpose of using bloom filter is to search faster, we can’t use slow hash functions, right? Cryptographic hash functions such as Sha-1, MD5 won’t be good choice for bloom filters as they are a bit slow. So, the better choices from the faster hash function implementations would be *murmur*, the *fnv* series of hashes, *Jenkins hashes* and *HashMix*.

## Applications

If the element is not in the bloom filter, then we know for sure we don’t need to perform the expensive lookup. On the other hand, if it is in the bloom filter, we perform the lookup, and we can expect it to fail some proportion of the time (the false positive rate).

For some more concrete examples:

*    You’ve seen in our given example that we could’ve use it to warn the user for weak passwords.

*    You can use bloom filter to prevent your users from accessing malicious sites.

*    Instead of making a query to an SQL database to check if a user with a certain email exists, you could first use a bloom filter for an inexpensive lookup check. If the email doesn’t exist, great! If it does exist, you might have to make an extra query to the database. You can do the same to search for if a ‘Username is already taken’.

*    You can keep a bloom filter based on the IP address of the visitors to your website to check if a user to your website is a ‘returning user’ or a ‘new user’. Some false positive value for ‘returning user’ won’t hurt you, right?
 
*   You can also make a Spell-checker by using bloom filter to track the dictionary words.

*    Want to know how Medium used bloom filter to decide if a user already read post? Read this mind-blowing, freaking awesome article about it.

### Applications of Bloom filters 

* Medium uses bloom filters for recommending post to users by filtering post which have been seen by user.

* Quora implemented a shared bloom filter in the feed backend to filter out stories that people have seen before.
    
* The Google Chrome web browser used to use a Bloom filter to identify malicious URLs

* Google BigTable, Apache HBase and Apache Cassandra, and Postgresql use Bloom filters to reduce the disk lookups for non-existent rows or columns

## Basic Python Implementation
```
# 3rd party
import mmh3

class BloomFilter(set):

    def __init__(self, size, hash_count):
        super(BloomFilter, self).__init__()
        self.bit_array = bitarray(size)
        self.bit_array.setall(0)
        self.size = size
        self.hash_count = hash_count

    def __len__(self):
        return self.size

    def __iter__(self):
        return iter(self.bit_array)

    def add(self, item):
        for ii in range(self.hash_count):
            index = mmh3.hash(item, ii) % self.size
            self.bit_array[index] = 1

        return self

    def __contains__(self, item):
        out = True
        for ii in range(self.hash_count):
            index = mmh3.hash(item, ii) % self.size
            if self.bit_array[index] == 0:
                out = False

        return out

def main():
    bloom = BloomFilter(100, 10)
    animals = ['dog', 'cat', 'giraffe', 'fly', 'mosquito', 'horse', 'eagle',
               'bird', 'bison', 'boar', 'butterfly', 'ant', 'anaconda', 'bear',
               'chicken', 'dolphin', 'donkey', 'crow', 'crocodile']
    # First insertion of animals into the bloom filter
    for animal in animals:
        bloom.add(animal)


    # Membership existence for already inserted animals
    # There should not be any false negatives
    for animal in animals:
        if animal in bloom:
            print('{} is in bloom filter as expected'.format(animal))
        else:
            print('Something is terribly went wrong for {}'.format(animal))
            print('FALSE NEGATIVE!')

    # Membership existence for not inserted animals
    # There could be false positives
    other_animals = ['badger', 'cow', 'pig', 'sheep', 'bee', 'wolf', 'fox',
                     'whale', 'shark', 'fish', 'turkey', 'duck', 'dove',
                     'deer', 'elephant', 'frog', 'falcon', 'goat', 'gorilla',
                     'hawk' ]
    for other_animal in other_animals:
        if other_animal in bloom:
            print('{} is not in the bloom, but a false positive'.format(other_animal))
        else:
            print('{} is not in the bloom filter as expected'.format(other_animal))

if __name__ == '__main__':
    main()
```

### Output is in the following:
```
dog is in bloom filter as expected
cat is in bloom filter as expected
giraffe is in bloom filter as expected
fly is in bloom filter as expected
mosquito is in bloom filter as expected
horse is in bloom filter as expected
eagle is in bloom filter as expected
bird is in bloom filter as expected
bison is in bloom filter as expected
boar is in bloom filter as expected
butterfly is in bloom filter as expected
ant is in bloom filter as expected
anaconda is in bloom filter as expected
bear is in bloom filter as expected
chicken is in bloom filter as expected
dolphin is in bloom filter as expected
donkey is in bloom filter as expected
crow is in bloom filter as expected
crocodile is in bloom filter as expected

badger is not in the bloom filter as expected
cow is not in the bloom filter as expected
pig is not in the bloom filter as expected
sheep is not in the bloom, but a false positive
bee is not in the bloom filter as expected
wolf is not in the bloom filter as expected
fox is not in the bloom filter as expected
whale is not in the bloom filter as expected
shark is not in the bloom, but a false positive
fish is not in the bloom, but a false positive
turkey is not in the bloom filter as expected
duck is not in the bloom filter as expected
dove is not in the bloom filter as expected
deer is not in the bloom filter as expected
elephant is not in the bloom, but a false positive
frog is not in the bloom filter as expected
falcon is not in the bloom filter as expected
goat is not in the bloom filter as expected
gorilla is not in the bloom filter as expected
hawk is not in the bloom filter as expected
```

## Explaining *Use case I*:

### Bloom Filter in Google's Webtable:

For instance, this is how Bloom filters are used in Google’s **Webtable** and Apache **Cassandra** that are among the most widely used distributed storage systems designed to handle massive amounts of data. Namely, these systems organize their data into a number of tables called Sorted String Tables (SSTs) that reside on the disk and are structured as key-value maps. In Webtable, keys might be website names, and values might be website attributes or contents. In Cassandra, the type of data depends on what system is using it, so for example, for Twitter, a key might be a User ID, and the value could be user’s tweets.

When users query for data, a problem arises because we do not know which table contains the desired result. To help locate the right table without checking explicitly on the disk, we maintain a dedicated Bloom filter in RAM for each of the tables, and use them to route the query to the correct table, in the way described in figure **Use case I** above.

In this example, we have 50 sorted string tables (SSTs) on disk, and each table has a dedicated Bloom filter that can fit into RAM due to its much smaller size. When a user does a lookup, the lookup first checks the Bloom filters. In this example, the first Bloom filter that reports the item as Present is Bloom filter No.3. Then we go ahead and check in the SST3 on disk whether the item is present. In this case, it was a false alarm. We continue checking until another Bloom filter reports Present. Bloom filter No.50 reports present, we go to the disk and actually locate and return the requested item.

## Explaining *Use case II*:

### Bloom Filters in Networks: Squid

Squid is a web proxy cache—a server that acts as a proxy between the client and other servers when the client requests a webpage, file, etc. Web proxies use caches to reduce web traffic, which means they maintain a local copy of recently-accessed links, in case they are requested again, and this usually enhances performance significantly. One of the protocols[6] designed suggests that a web proxy locally keeps a Bloom filter for each of its neighboring servers’ cache contents. This way when a proxy is looking for a webpage, it first checks its local cache. If a cache miss occurs locally, the proxy checks all its Bloom filters to see whether any of them contain the desired webpage, and if yes, it tries to fetch the webpage from the neighbor associated with that Bloom filter instead of directly fetching the page from the Web.

Squid implements this functionality and it calls Bloom filters Cache Digests (see Figure Use case II) Because data is highly dynamic in the network scenario, and Bloom filters are only occasionally broadcasted between proxies, false negatives can arise.

Web proxies keep the copies of recently-accessed web pages, but also keep the record of recently accessed web pages of their neighbors by having each proxy occasionally broadcast the Bloom filter of their own cache. In this example, a user requests a web page x, and a web proxy A cannot find it in its own cache, so it queries the Bloom filters of B, C and D. The Bloom filter of D reports Found/Present for x, so the request is forwarded to D. Note that, because Bloom filters are not always up-to-date, and the network environment is highly dynamic, by the time we get to the right proxy, the cache might have deleted the resource that we are looking for. Also, false negatives may arise due to the gap in the broadcasting times.

## Use case III:

### Bitcoin mobile app

Peer-to-peer networks use Bloom filters to communicate data, and a well-known example of that is Bitcoin. An important feature of Bitcoin is ensuring transparency between clients, i.e., each node should be able to see everyone’s transactions. However, for nodes that are operating from a smartphone or a similar device of limited memory and bandwidth, keeping the copy of all transactions is highly impractical. This is why Bitcoin offers the option of simplified payment verification (SPV), where a node can choose to be a light node by advertising a list of transactions it is interested in. This is in contrast to full nodes that contain all the data

Light nodes compute and transmit a Bloom filter of the list of transactions they are interested in to the full nodes. This way, before a full node sends information about a transaction to the light node, it first checks its Bloom filter to see whether a node is interested in it. If the false positive occurs, the light node can discard the information upon its arrival.

Read more here: https://freecontent.manning.com/all-about-bloom-filters/
Ref: https://hackernoon.com/probabilistic-data-structures-bloom-filter-5374112a7832

