## Concept 1 — Stranded Power
Stranded Power = difference between rated capacity and usable capacity

- Nameplate rating = maximum it can draw
- Actual draw varies
	- 50-70% for traditional servers
	- Reserved for unused. This is not vSphere reservation. It means, it is available to use
	- So this unused powered is stranded power
- AI workload, actual draw is 100%, hence if you are mixing AI workload with traditional workload, it poses huge oversubscription risk for power and cooling
- Always note a difference between structurally stranded and operational stranded.
	- Structurally stranded - applies 80% derating rule
	- operationally stranded assumes 80% derating rule is followed
	- 32A x 80% = 25.6A available for use. 20% is structurally stranded.
	- Now out 25.6 A, if servers are using 12A, the 25.6 - 12 = 13.6A is operationally stranded. In other words, 13.6A is available for further use
- Power is allocated in circuit units, so when we say 32A it ends up 17.48 KW
	- 240V x 32A x 80% x .95 = 17.48 KW
	- But if DGX boxes is only drawing 1.7 KW per outlet, then 17.48 - 1.7 = 15.78 KW is per circuit power available or stranded.
### SA Angle
Availability of capacity is purely a baseline information but the real metric is whether it is
- named or rated capacity
- meter capacity / visible on Rack
- then one can find the available capacity or head room in existing DC


## Concept 2 — CRAC vs CRAH

### CRAC - Computer Room Air Condition

- Like Air condition in home, 
	- it is self contained, plug it in, it cools
	- air is passed over refrigerant coils
- Compressor consumes more power, hence less efficient
- serve edge use cases, small DC.
- range is 10 - 50 KW per unit

### CRAH - Computer Room Air Handler
- no compressor, less power and efficient.
- only fan and a water coil.
- Requires external facility chiller water supply from a chiller plant
- Range 50 - 500+ KW per unit, suitable large DC

### Can CRAC or CRAH server DGX?
- DGX needs a temperature between 18 - 27 degree Celsius
- 4 DGX box need 40.8 KW, but cooling perspective one need to ensure CRAH unit run at max 80%, living 20% head room for peaks and emergency situation and if you have no containment in your DC facility, efficiency drops to 60-70%, so we are looking for at least 100 KW i.e. it runs at max 80 KW and with no containment, 60% efficiency worst case, it get 52 KW which meets 40.8 KW requirement. With containment the expected efficiency is 90-95%
- Also we need two CRAC N+1 configuration.

### SA Angle
What is the cooling capacity per KW is available?  and then comes the question whether it is CRAC or CRAH. Availability of facility chilling water or CRAH will help you if the cooling capacity can be scaled. Even though one has 50 KW cooling capacity available without containment the cooling efficiency drops to 60 to 70%, do you have containment in place

## Concept 3 — Capacity Planning: The Four Questions

### What is the rated power capacity per rack?
This question get easily answered e.g. if it is 20 KW, you simply 20 x 80% = 16 KW
16 KW is what is available and now see how much is used.

### What is the actual current draw per rack?
This is only possible if they per outlet level monitoring or inlet level monitoring. So get the number and check what is available by subtracting operational stranded to what is used by per Rack level

### What is your current cooling capacity in KW?
This we ask always in KW, because we need to understand how much of the heat generated in KW is taken off by our cooling system. If answer is 500KW, multiple by 80% i.e. 400 KW is max that can be used.

#### Example

Rated Power per Rack = 20 KW
Apply derating = 20 KW x 80% = 16 KW
Actual available = 16 KW
 based on Inlet metering Rack is currently drawin= 6 KW
Stranded 16 - 6 = 10 KW stranding or available to use.
But then does one DGX boxes fits into this? No, because it needs 10.2 KW

## Concept 4 — The Cooling Oversubscription Problem for AI

When the cooling demand is greater than cooling capacity, then oversubscription risk increases. This is especially true with AI Workload mixed with Non-AI workload

### Example

Rated Capacity per = 20 KW
At 80% derated = 16 KW (20 * 80%)

For server workload, average utilization 50% of the actual requirement (e.g. 5 KW), so per Rack 11 KW available.
One DGX Box fits, it runs at peak all the time

5KW + 11 KW = 16 KW
Now server workload starts peaking, 5 KW becomes 10 KW, circuit breaker limit reached already.

### SA Angle
spread nodes across the racks (spread density) , upgrade circuit capacity or assess co-location
