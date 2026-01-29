

>Hi I'm shanks i played as RubyDaCherry and was able to get 19th during the comp.My main areas are Web and OSINT . Here are the OSINT challs . Ill do a writeup for the web challenges later.


## **1. Bruhh!! It’s CHISHIYA**

**Difficulty:** Easy  
**Points:** 100  
**Solves:** 34

**Username Provided:** `shuntaro_.chishiya_`

This challenge was solved using a simple Google Dork:

`"shuntaro_.chishiya_"`

![](images/chishiya1.png)

Searching this query led directly to the target account. Upon accessing the account, the flag was visible in the first image.

![](images/chishiya2.png)


**Flag:**

`NICCTF26{but_1m_cl3v3r}`

---

## **2. Finding Beagle**

**Difficulty:** Easy  
**Points:** 200  
**Solves:** 58

**Provided Image:** 
![](images/location.jpg)

### Method

1. Performed a reverse image search on the provided image.
2. The search returned exact matches.
    ![](images/beagle2.png)

3. The location was identified as **Dehradun Ghantaghar**.
    
4. Using Google Maps, nearby pet shops were examined.
    ![](beagle3.png)

5. The relevant shop(as shown) was found, along with its contact number:  
    `+91 7668669112`
    

**Flag:**

`NICCTF26{7668669112}`

---

## **3. Cockpit Climb**

**Difficulty:** Easy  
**Points:** 200  
**Solves:** 80

**Provided Image:** ![](images/c.jpeg)


### Method

A reverse image search was conducted on the provided image. This revealed that the aircraft in the image was an **F-104 Starfighter**.

![](images/cockpit2.png)


**Flag:**

`NICCTF26{F-104_Starfighter}`

---

## **4. Long Distance Friend**

**Difficulty:** Easy  
**Points:** 100  
**Solves:** 95

**Provided Image:** 
![[](images/friend.jpg)
### Method

A reverse image search was performed on the image, which directly led to the required information.
![](friend2.png)

**Flag:**

`NICCTF26{Rustomjee_Seasons}`

---

## **5. Overhang**

**Difficulty:** Hard  
**Points:** 200  
**Solves:** 1

**Provided Image:** 
![](image8.png)


> For this challenge  I was only able to solve half during the  competition and  later upsolved it all after competition ended with assistance from **0xfun**.

---

### **Step 1: Identifying the Author**

The username `IncredibleZuess` was searched on Google:

`"IncredibleZuess"`

This search led to an OpenStreetMap account, which revealed that the user was affiliated with **North-West University, South Africa**.
![](overhang2.png)


---

### **Step 2: Locating the Climbing Wall**

A targeted search was performed:

`"North Western University south africa" AND "Wall Climbing"`
![](overhang3.png)

This returned relevant results pointing to a climbing facility at the university. Further investigation revealed the wall number:
![](overhang4.png)


```yaml
G16 
```

Searching for `NWU G16` and locating the Faculty of Health Sciences on Google Maps helped identify the precise location.
![](overhang5.png)

---

### **Step 3: Analyzing the Flag Format**

The flag format was:

`NICCTF26{NameOfWall_NameOfPreviousClubOwner_BuildingName}`

Therefore, three components were required:

1. Name of the wall
    
2. Name of the previous club owner
    
3. Name of the building
    

---

### **Step 4: Using Hints and Wayback Machine**

Hint 2 suggested using the Wayback Machine.

The climbing club’s website was checked on web archives. A discrepancy in the club ownership information was observed, revealing the **previous club owner’s name**.

This solved the second component.
![](overhangwebarchive.png)

---

### **Step 5: Using Instagram**

Hint 5 mentioned the Instagram account:

`nwurockclimbing`

This account was reviewed to find posts featuring the wall. Based on Hint 3, the route name was identified as the wall name:

`Eben`

This solved the first component.
![](overhangIG.png)

---

### **Step 6: Identifying the Building Name**

The final step was to determine the building where the wall was located. By examining Google Maps and campus locations, the building was identified as:

`Pharmacy`

(Although this detail was not entirely certain, it was the most consistent result.)
![](images/pharm.png)

---

### **Final Flag**

`NICCTF26{Eben_JapieVenter_Pharmacy}`