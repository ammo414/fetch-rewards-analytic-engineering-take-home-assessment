Below are the data quality concerns that I've found with the given JSON files. I've also included some things that I thought might be a concern but turned out weren't. All research was done with either Bash or Python run on the JSON files themselves -- I did not transform the data at this point. My code is inserted below and also uploaded in this directory. The other .md in this directory, findings-email.md, will have the results formatted and described for a product or business leader not familiar with my work.


# Sparsely Populated Joinable Keys

## Barcodes
As part of the schema that I designed, I decided to join Brands[barcode] to Receipts[rewardsReceiptItemList][barcode]. While that makes sense conceptually, it remains to be seen if the barcodes actually match up often for this to be a meaningful join once we load this data into a database. I wrote the following python script to confirm.

```Python
import json


def load_data(filename):
    data = []
    with open(filename) as receipts_data:
        for line in receipts_data:
            data.append(json.loads(line))
    return json.loads(json.dumps(data))  # python can't handle multiple objects at once normally.
    # this is a quick workaround

def joining_on_barcodes(receiptsLoad, brandsLoad):
    RIBarCodesJoinable = set()
    RIBarCodesTotal = set()
    for r in receiptsLoad:
        if 'rewardsReceiptItemList' in r:
            for ri in r['rewardsReceiptItemList']:
                try:
                    receiptsBC = ri['barcode']
                    RIBarCodesTotal.add(receiptsBC)
                except KeyError:
                    receiptsBC = None
            for b in brandsLoad: #n^2, maybe we can do something better here?
                try:
                    brandBC = b['barcode']
                    if receiptsBC == brandBC:
                        RIBarCodesJoinable.add(receiptsBC)
                        break
                except KeyError:
                    brandBC = None

    print(len(RIBarCodesJoinable))
    print(len(RIBarCodesTotal))

    BBarCodesJoinable = set()
    BBarCodesTotal = set()
    for b in brandsLoad:
        try:
            brandBC = b['barcode']
            BBarCodesTotal.add(brandBC)
        except KeyError:
            brandBC = None
        for r in receiptsLoad:
            if 'rewardsReceiptItemList' in r:
                for ri in r['rewardsReceiptItemList']:
                    try:
                        receiptsBC = ri['barcode']
                        if brandBC == receiptsBC:
                            BBarCodesJoinable.add(brandBC)
                            break
                    except KeyError:
                        receiptsBC = None

    print(len(BBarCodesJoinable))
    print(len(BBarCodesTotal))

#output:
# 0
# 568
# 16
# 1160
```
Of the 568 unique barcodes present in receipts.json, none of them can be joined on, and likewise only 16 out of 1160 barcodes present in brands.json can be joined on. As I show later on, this is likely because most of brands.json is test data rather than real data, which naturally means it won't be related to the real data that is present in receipts.json. However, this issue does make validation at this point in time difficult, as for example SQL queries related to brands data will fail to give any meaningful results. Its worth noting that there are 7 duplicate barcodes in brands.json, but again since this data is so bad I don't think further research is going to be particularly helpful until we can figure out what is going on with brands.json as a whole. 

## userIDs and duplicate data
A similar script can be run on receipts.userId and users._id:

```python

import json

# load receipts data
def load_data(filename):
    data = []
    with open(filename) as receipts_data:
        for line in receipts_data:
            data.append(json.loads(line))
    return json.loads(json.dumps(data))  # python can't handle multiple objects at once normally.
    # this is a quick workaround

def joining_on_users(receiptsLoad, usersLoad):
    usersIDTotal = set()
    usersIDJoinable = set()
    for u in usersLoad:
        userID = u['_id']['$oid']
        usersIDTotal.add(userID)
        for r in receiptsLoad:
            if 'userId' in r:
                receiptsUser = r['userId']
                if receiptsUser == userID:
                    usersIDJoinable.add(userID)
                    break

    print(len(usersIDJoinable))
    print(len(usersIDTotal))

    receiptUserTotal = set()
    receiptUserJoinable = set()
    for r in receiptsLoad:
        if 'userId' in r:
            receiptsUser = r['userId']
            receiptUserTotal.add(receiptsUser)
            for u in usersLoad:
                userID = u['_id']['$oid']
                if userID == receiptsUser:
                    receiptUserJoinable.add(receiptsUser)
                    break

    print(len(receiptUserJoinable))
    print(len(receiptUserTotal))

# output
# 141
# 212
# 141
# 258
```

These results are a bit more reasonable, but still not great. Of the 212 unique users in receipts.json, 141 of them can be joined on, and likewise of the 258 unique users in users.json. One thing that I accidentally discovered here is that there is a lot of duplicate data in users.json: a `wc -l users.json` in Bash tells us that there are 495 lines in users.json, and we know each line represents its own user. However, the above script shows us there are 258 unique oids in the file. That means that almost half of the data is duplicate, which is pretty bad. Further research would need to be done to check if the entire row is duplicative, or if IDs are being reused for multiple sets of demographics. However, that kind of investigation would best be done in SQL or with another tool that lets us easily group data, so I won't do it here. The query would look something like the below, in mySQL:

```mysql
SELECT _id, count(createdDate), count(active), count(role), count(state) 
FROM users
GROUP BY _id
HAVING count(createdDate) >1 OR count(active) >1 OR count(role) >1 OR count(state) >1 
```
This should find any user with multiple active values, or roles, or lives in multiple states, or whatever other duplicate values we care about. If all we are dealing with is completely duplicative data, then that isn't too big a deal since the IDs are the same as well, so the database should automatically ignore them during the loading process. However, if there are differences in demographic data, I would have to know what the intention there is: are we supposed to treat later rows as updates to previous rows with the same ID? If so, then we can potentially scrub out previous rows and only load the last one for each ID, if saving past demographics is not needed. This would help with performance during the load into the database.

# Concerns with Reciepts

## Missing data keys in the schema

While the schema provided gives us information about most of the keys in the data, the nested JSON object in rewardsReceiptItemList was undocumented. In order to find out what keys existed in this object, I ran the following
Python script:

```python
import json

data = []
with open('receipts.json') as receipts_data:
    for line in receipts_data:
        data.append(json.loads(line))
receiptsLoad = json.loads(json.dumps(data))  # python can't handle multiple objects at once normally. this is a quick workaround

ReceiptItemList = set()
for x in receiptsLoad:
    for key in x:
        if key == 'rewardsReceiptItemList':
            for itemList in x[key]:
                for key2 in itemList:
                    ReceiptItemList.add(key2)

print(ReceiptItemList)

#{'finalPrice', 'originalMetaBriteDescription', 'brandCode', 'itemPrice', 'needsFetchReviewReason', 'description', 'partnerItemId', 'barcode', 'targetPrice', 'originalReceiptItemText', 'needsFetchReview', 'competitiveProduct', 'preventTargetGapPoints', 'pointsPayerId', 'metabriteCampaignId', 'itemNumber', 'userFlaggedQuantity', 'userFlaggedNewItem', 'userFlaggedPrice', 'originalMetaBriteItemPrice', 'competitorRewardsGroup', 'deleted', 'userFlaggedDescription', 'userFlaggedBarcode', 'rewardsGroup', 'priceAfterCoupon', 'rewardsProductPartnerId', 'discountedItemPrice', 'originalMetaBriteQuantityPurchased', 'pointsEarned', 'pointsNotAwardedReason', 'quantityPurchased', 'originalMetaBriteBarcode', 'originalFinalPrice'}

```

As I described in task 1, there is no guarantee that ths is the full set of keys over all the data, but for the sake of this exercise I am operating under the assumption that it is. While I was able to document the names of the headers, I still do not know what some of them represent. For example, `deleted` or `pointsPayerId` is a little ambigious. I would need someone with domain knowledge to fill in those gaps for me.

## Duplicate data? barcode vs originalMetaBarcode
Within receipts[ReceiptItemList], there are two keys that seems similar at first glance: `barcode` and `originalMetaBriteBarcode`. Manually going through the JSON, it seemed like if there was an `originalMetaBriteBarcode`, then it was the same as `barcode`. I figured that, even if the two barcodes aren't the same for a given item, if there is a one to one relationship between barcodes and MetaBriteBarcodes, then we can consider MetaBriteBarcodes duplicate data and store it elsewhere. If we can do that, then there would be many fewer writes on ReceiptItems whenever we load data, greatly increasing performance of our loads. I wrote the following Python script to see if that was true:

```python
import json

data = []
with open('receipts.json') as receipts_data:
    for line in receipts_data:
        data.append(json.loads(line))
receiptsLoad = json.loads(json.dumps(data))  # python can't handle multiple objects at once normally. this is a quick workaround

# duplicate barcode data
barcode_dict = {}
for x in receiptsLoad:
    if 'rewardsReceiptItemList' in x:
        for y in x['rewardsReceiptItemList']:
            if 'barcode' in y:
                barcode = y['barcode']
                if barcode not in barcode_dict:
                    barcode_dict[barcode] = None
                if 'originalMetaBriteBarcode' in y:
                    MBbarcode = y['originalMetaBriteBarcode']
                    if (barcode_dict[barcode] is None or
                            barcode_dict[barcode] == [MBbarcode]):  # wrapping with a list is a quick and dirty way to
                        barcode_dict[barcode] = [MBbarcode]         # potentially append multiple values if needed
                    else:
                        barcode_dict[barcode].append(MBbarcode)
                        print(f'{barcode} has multiple MBbarcodes.')
for key in barcode_dict:
    print(key, barcode_dict[key])

# a truncated output:
# 4011 has multiple MBbarcodes.
# 4011 has multiple MBbarcodes.
# 4011 has multiple MBbarcodes.
# 4011 has multiple MBbarcodes.
# 4011 ['028400642255', '', '', '028400642255', '028400642255']
# 028400642255 None
# 1234 None
# 046000832517 None
# 013562300631 None
# 034100573065 ['034100573065']
# 075925306254 ['075925306254']
```
The only barcode associated with multiple originalMetaBriteBarcodes is `4011`. If we can figure out what is so special about `4011`, then we can create a lookup table to store each `originalMetaBriteBarcode` and its associated barcode, and eliminate a column from `ReceiptItems`, as the relationship between the two keys is otherwise at most one-to-one.

### What is so special about 4011?
Once again, I took a look manually and found that a `barcode` of `4011` was often associated with a description of `ITEM NOT FOUND`. It seems like `4011` is a placeholder barcode. I used the following Bash script to see if that is true.

```bash
grep -E '4011","description":.{20}' -o receipts.json

# searches for strings in receipts.json that matches the following regular expression pattern:
# the string '"4011", description":' followed by any 20 characters. We are returning just the matched text rather than the entire line that the matched text is part of.
```

Unfortunately here, my hypothesis was not true. Here's a snippet of some of the results:

```
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"BANANAS","discounte
4011","description":"BANANAS","discounte
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"ITEM NOT FOUND","fi
4011","description":"Yellow Bananas","di
4011","description":"Yellow Bananas","di
4011","description":"Yellow Bananas","di
4011","description":"Yellow Bananas","di
4011","description":"Yellow Bananas","di
4011","description":"ITEM NOT FOUND","fi
```
This barcode is also used for bananas and yellow bananas. Its still strange to me that the only descriptions in the dataset are 'Yellow Bananas', 'BANANAS', and 'ITEM NOT FOUND'. I would like to do some further research into what this barcode is meant to represent, as this data now seems pretty shoddy, but I unfortunately don't think the answers to my question are in the data itself. Instead, I would have to talk to a barcode expert to see if they know what is going on here. (Or I can Google '4011 barcode' and get the answer, but I still think that meeting up with an expert is the correct way to go for these kinds of things.)

## If Item Not Found, what do we do next?
Speaking of `ITEM NOT FOUND`, it sounds like this data point should essentially flag the row for manual review so that the item can eventually be found. 'Receipts[rewardsReceiptItemList][needsFetchReview]' looks like it acts as that flag. I would expect this value to be true for any item that has a description of 'ITEM NOT FOUND'. Unfortunately, that is not the case:

```bash 
grep -E 'ITEM NOT FOUND".+needsFetchReview":true\}' receipts.json

# searches for strings that match the following pattern:
# the string 'ITEM NOT FOUND".' followed by any number of characters followed by the string 'needsFetchReview":true}'
```
returns no results. This seems like a gap to me, and I think that any time we have an 'ITEM NOT FOUND' or something similar, we should be manually reviewing those items in hopes of reducing the overall number of 'ITEM NOT FOUND's that we see.

## The dates are good, but I'm glad I checked
I noticed that the dates are stored as integers and not as ISO dates. I guessed that they were Linux timestamps and was fortunately correct. I figured it would be a good idea to confirm that none of the dates are particularly weird: nothing from too long ago or in the future. I wrote the following bash one-liner to confirm:

```bash
grep -E '"\$date":[0123456789]{13}' -o receipts.json | grep -E [0123456789]{10} -o | while read line; do date -d '@'$line; done > receiptsEpochTime'

# searches for strings that are just 13 digits, pulls the first 10 of those digits (I don't care about the milliseconds), adds an '@' in front of the number for the sake of formatting, transforms them into human readable dates via the GNU program, date, and pipes them into a file named receiptsEpochTime.
```

With that `receiptsEpochTime`, I'm able to quickly `grep` for any year I want to search for, say 2020, with a `grep '2020' receiptsEpochTime`. I've confirmed that all dates are within 2017 and 2021, although there are very few 2017s compared to the rest.


## Values are correctly typed and incoming keys are correctly formatted
An issue I've seen before are values being a number of different data types, and formatting issues or typos with keys. Both can be checked at once with the following Python code:

```Python
# ensure headers are correctly formatted and values are correctly typed
receipts_headers = {'_id': str,
                    'bonusPointsEarned': int,
                    'bonusPointsEarnedReason': str,
                    'createDate': int,
                    'dateScanned': int,
                    'finishedDate': int,
                    'modifyDate': int,
                    'pointsAwardedDate': int,
                    'pointsEarned': int,
                    'purchaseDate': int,
                    'purchasedItemCount': int,
                    'rewardsReceiptItemList': list,
                    'rewardsReceiptStatus': str,
                    'totalSpent': float,
                    'userId': str}

errorString = 'type error with '

for x in receiptsLoad:
    for key in x:
        if key not in receipts_headers:
            print('unexpected header')
        if key == '_id':
            if type(x[key]['$oid']) is not str:
                print(errorString + key)
        elif 'DATE' in key.upper():
            if type(x[key]['$date']) is not int:
                print(errorString + key)
        elif key == 'totalSpent' or key == 'pointsEarned':
            try:
                i = float(x[key])  # today I learned why decimals are stored as strings in JSON
            except ValueError:
                print(errorString + key)
        elif type(x[key]) is not receipts_headers[key]:
            print(errorString + key)
```

Pretty much everything looks good here so I'm not going to harp on anything for too long, but like I mentioned in task 1, I'm not sure if data points like `totalSpent` or `pointsEarned` should remain as strings/varchars or if they should be converted to numerics. 

# Concerns with Brands
## There are a lot of test brands
Looking manually at the JSON, we're immediately greeted with the word "test" everywhere on the screen, which seems wrong. We can find the total line count of the file with a `wc -l brands.json` and see that there are 1167 lines, and so, assuming one brand per line, that there are 1167 brands in the file. If we do a very naive search and line count of just the phrase 'test brand', using `grep 'test brand' brands.json | wc -l`, we get 428 results. That means that at least a third of our brands are test brands. While such data can be useful in testing and development situations, it shouldn't be entering production databases. Until we've investigated where this data is coming from, I would not load this data into a data warehouse.

# Concerns with Users
## Are they test users?
Since PII has been scrubbed from this file, and since test user records are usually identified by their name, we cannot from this data alone tell if there are test users in our file. However, it is something that should be looked into if at all possible, given our issues with Brands.

## There are non-consumers in the data
According to the Users data schema, role is a "constant value set to 'CONSUMER'". We can quickly `grep` to see if that is true:

```bash
grep -E 'role":"[^c]' -i users.json

#looks for strings of the following pattern:
# 'role":" followed by any character that is not 'c' or 'C'
```
The following truncated snippet shows that we have a number of fetch staff in the data as well:

```
{"_id":{"$oid":"59c124bae4b0299e55b0f330"},"active":true,"createdDate":{"$date":1505830074302},"lastLogin":{"$date":1612802578117},"role":"fetch-staff","state":"WI"}
{"_id":{"$oid":"59c124bae4b0299e55b0f330"},"active":true,"createdDate":{"$date":1505830074302},"lastLogin":{"$date":1612802578117},"role":"fetch-staff","state":"WI"}
{"_id":{"$oid":"59c124bae4b0299e55b0f330"},"active":true,"createdDate":{"$date":1505830074302},"lastLogin":{"$date":1612802578117},"role":"fetch-staff","state":"WI"}
{"_id":{"$oid":"59c124bae4b0299e55b0f330"},"active":true,"createdDate":{"$date":1505830074302},"lastLogin":{"$date":1612802578117},"role":"fetch-staff","state":"WI"}
```
The data description contradicts what is actually going on in the data, which means that either the schema description needs to be updated or the data is wrong. We would need to talk to business to determine which is the case, but it wouldn't be too hard of a fix either way. If the data is wrong, we would simply need to filter by role before loading this data into the warehouse. If the description is wrong, we need to document all possible values of `role`.


## The dates are still good, and I'm still glad I checked
There are two date-related things that I can potentially see go wrong here. The first is the same as what we checked in Receipts: are there any weird dates in the file? The second is whether any 'lastLogin' are before the 'created-Date'. Both are easy enough to check for.

We can reuse my bash one-liner on this file and similarly grep for any weird years:

```bash
grep -E '"\$date":[0123456789]{13}' -o users.json | grep -E [0123456789]{10} -o | while read line; do date -d '@'$line; done > usersEpochTime
```

Nothing particluarly weird, which is good. I looked for dates 2013 or earlier, as that is when Fetch was founded, and of course for dates in the future.

I wrote the following Python script to check for `lastLogins` earlier than `created-Date`:

```python
data = []
with open('users.json') as users_data:
    for line in users_data:
        data.append(json.loads(line))
usersLoad = json.loads(json.dumps(data))  # python can't handle multiple objects at once normally. this is a quick workaround

for x in usersLoad:
    createdDate = None
    lastLogin = None
    for key in x:
        if key == 'createdDate':
            createdDate = x[key]['$date']
        if key == 'lastLogin':
            lastLogin = x[key]['$date']
    if createdDate is not None and lastLogin is not None:
        if lastLogin - createdDate < 0:
            print(x['_id'])
```

This gets the integers representing last login and created Date, and if a user's last login is smaller than created Date, then it prints the ID of that user. Nothing was printed out, which means we are in the clear.

