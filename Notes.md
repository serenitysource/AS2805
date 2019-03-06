Foreword

Previously I briefly touched on the AS2805 standards, and now I have an implementation of the parsing of these messages

The full code of this post is available here : https://github.com/Arthurvdmerwe/AS2805_Python_Implementation

I have a C# implementation of this as well here:

https://github.com/Arthurvdmerwe/AS2805

The code is thanks to the following author: http://www.vulcanno.com.br/python/ISO8583.html

He has a brilliant implementation of all the structures of the 8583 format, which goes hand in hand with AS2805

First off, the AS2805 standard includes a range of specifications, from Key handling settlement. This is not a post to explain all the details, as the key and settlement handling is a processor specific implementation.

AS2805 is extremely similar to the ISO8583 specification, and most of it can be taken directly out of this specification.

AS2805 Financial transaction card originated messages — Interchange message specifications is the International Organization for Standardization standard for systems that exchange electronic transactions made by cardholders using payment cards. It has three parts:

    Part 1: Messages, data elements and code values
    Part 2: Application and registration procedures for Institution Identification Codes (IIC)
    Part 3: Maintenance procedures for messages, data elements and code values

Introduction

A card-based transaction typically travels from a transaction acquiring device, such as a point-of-sale terminal or an automated teller machine (ATM), through a series of networks, to a card issuing system for authorization against the card holder’s account. The transaction data contains information derived from the card (e.g., the account number), the terminal (e.g., the merchant number), the transaction (e.g., the amount), together with other data which may be generated dynamically or added by intervening systems. The card issuing system will either authorize or decline the transaction and generate a response message which must be delivered back to the terminal within a predefined time period.

AS2805 defines a message format and a communication flow so that different systems can exchange these transaction requests and responses. The vast majority of transactions made at ATMs use AS2805 at some point in the communication chain, as do transactions made when a customer uses a card to make a payment in a store (EFTPOS). In particular, both the MasterCard andVisa networks base their authorization communications on the ISO 8583 standard, as do many other institutions and networks. AS2805 has no routing information, so is sometimes used with aTPDU header.

Cardholder-originated transactions include purchase, withdrawal, deposit, refund, reversal, balance inquiry, payments and inter-account transfers. AS2805 also defines system-to-system messages for secure key exchanges, reconciliation of totals, and other administrative purposes.

Report this ad

Although AS2805 defines a common standard, it is not typically used directly by systems or networks. It defines many standard fields (data elements) which remain the same in all systems or networks, and leaves a few additional fields for passing network specific details. These fields are used by each network to adapt the standard for its own use with custom fields and custom usages.

The placements of fields in different versions of the standard varies;

An AS2805 message is made of the following parts:

    Message type indicator (MTI)
    One or more bitmaps, indicating which data elements are present
    Data elements, the fields of the message

 
Message type indicator

This is a 4 digit numeric field which classifies the high level function of the message. A message type indicator includes the ISO 8583 version, the Message Class, the Message Function and the Message Origin, each described briefly in the following sections. The following example (MTI 0110) lists what each digit indicates:

  0xxx -> version of AS2805 
  x1xx -> class of the Message (Authorization Message)
  xx1x -> function of the Message (Request Response)
  xxx0 -> who began the communication (Acquirer)

AS2805 version

Position one of the MTI specifies the versions of the AS2805 standard which is being used to transmit the message.
Position 	Meaning
0xxx 	ISO 8583-1:1987 version
1xxx 	ISO 8583-2:1993 version
2xxx 	ISO 8583-1:2003 version
3xxx 	Reserved for ISO use
4xxx 	Reserved for ISO use
5xxx 	Reserved for ISO use
6xxx 	Reserved for ISO use
7xxx 	Reserved for ISO use
8xxx 	Reserved for National use
9xxx 	Reserved for Private use
Message class

Position two of the MTI specifies the overall purpose of the message.
Position 	Meaning 	Usage
x1xx 	Authorization Message 	Determine if funds are available, get an approval but do not post to account for reconciliation, Dual Message System (DMS), awaits file exchange for posting to account
x2xx 	Financial Messages 	Determine if funds are available, get an approval and post directly to the account, Single Message System (SMS), no file exchange after this
x3xx 	File Actions Message 	Used for hot-card, TMS and other exchanges
x4xx 	Reversal Message 	Reverses the action of a previous authorization
x5xx 	Reconciliation Message 	Transmits settlement information message
x6xx 	Administrative Message 	Transmits administrative advice. Often used for failure messages (e.g. message reject or failure to apply)
x7xx 	Fee Collection Messages 	
x8xx 	Network Management Message 	Used for secure key exchange, logon, echo test and other network functions
x9xx 	Reserved by ISO 	
Message function

Position three of the MTI specifies the message function which defines how the message should flow within the system. Requests are end-to-end messages (e.g., from acquirer to issuer and back with timeouts and automatic reversals in place), while advices are point-to-point messages (e.g., from terminal to acquirer, from acquirer to network, from network to issuer, with transmission guaranteed over each link, but not necessarily immediately).
Position 	Meaning
xx0x 	Request
xx1x 	Request Response
xx2x 	Advice
xx3x 	Advice Response
xx4x 	Notification
xx8x 	Response acknowledgment
xx9x 	Negative acknowledgment
Message origin

Position four of the MTI defines the location of the message source within the payment chain.
Position 	Meaning
xxx0 	Acquirer
xxx1 	Acquirer Repeat
xxx2 	Issuer
xxx3 	Issuer Repeat
xxx4 	Other
xxx5 	Other Repeat
Examples

Bearing each of the above four positions in mind, an MTI will completely specify what a message should do, and how it is to be transmitted around the network. Unfortunately, not all AS2805 implementations interpret the meaning of an MTI in the same way. However, a few MTIs are relatively standard:
MTI 	Meaning 	Usage
0100 	Authorization request 	Request from a point-of-sale terminal for authorization for a cardholder purchase
0110 	Issuer Response 	Issuer response to a point-of-sale terminal for authorization for a cardholder purchase
0120 	Authorization Advice 	When the Point of Sale device breaks down and you have to sign a voucher
0121 	Authorisation Advice Repeat 	If the advice times out
0130 	Issuer Response to Authorization Advice 	Confirmation of receipt of authorization advice
0200 	Acquirer Financial Request 	Request for funds, typically from an ATM or pinned point-of-sale device
0210 	Issuer Response to Financial Request 	Issuer response to request for funds
0220 	Acquirer Financial Advice 	e.g. Checkout at a hotel. Used to complete transaction initiated with authorization request
0221 	Acquirer Financial Advice repeat 	If the advice times out
0230 	Issuer Response to Financial Advice 	Confirmation of receipt of financial advice
0400 	Acquirer Reversal Request 	Reverses a transaction
0420 	Acquirer Reversal Advice 	Advises that a reversal has taken place
0421 	Acquirer Reversal Advice Repeat Message 	If the reversal times out
0430 	Issuer Reversal Response 	Confirmation of receipt of reversal advice
0800 	Network Management Request 	Echo test, logon, log off etc.
0810 	Network Management Response 	Echo test, logon, log off etc.
0820 	Network Management Advice 	Keychange
Bitmaps

Within AS2805, a bitmap is a field or subfield within a message which indicates which other data elements or data element subfields may be present elsewhere in a message.

A message will contain at least one bitmap, called the Primary Bitmap which indicates which of Data Elements 1 to 64 are present. A secondary bitmap may also be present, generally as data element one and indicates which of data elements 65 to 128 are present. Similarly, a tertiary, or third, bitmap can be used to indicate the presence or absence of fields 129 to 192, although these data elements are rarely used.

The bitmap may be transmitted as 8 bytes of binary data, or as 16 hexadecimal characters 0-9, A-F in the ASCII or EBCDIC character sets.

A field is present only when the specific bit in the bitmap is true. For example, byte ’82x is binary ‘1000 0010’ which means fields 1 and 7 are present in the message and fields 2, 3, 4, 5, 6, and 8 are not present.

Examples —–
Bitmap 	Defines presence of
4210001102C04804 	Fields 2, 7, 12, 28, 32, 39, 41, 42, 50, 53, 62
7234054128C28805 	Fields 2, 3, 4, 7, 11, 12, 14, 22, 24, 26, 32, 35, 37, 41, 42, 47, 49, 53, 62, 64
8000000000000001 	Fields 1, 64
0000000000000003
(secondary bitmap) 	Fields 127, 128

Explanation of Bitmap (8 BYTE Primary Bitmap = 64 Bit) field 4210001102C04804
BYTE1 : 01000010 = 42x (counting from the left, the second and seventh bits are 1, indicating that fields 2 and 7 are present)
BYTE2 : 00010000 = 10x (field 12 is present)
BYTE3 : 00000000 = 00x (no fields present)
BYTE4 : 00010001 = 11x (fields 28 and 32 are present)
BYTE5 : 00000010 = 02x (field 39 is present)
BYTE6 : 11000000 = C0x (fields 41 and 42 are present)
BYTE7 : 01001000 = 48x (fields 50 and 53 are present)
BYTE8 : 00000100 = 04x (field 62 is present)

0________10________20________30________40________50________60__64
1234567890123456789012345678901234567890123456789012345678901234  n-th bit
0100001000010000000000000001000100000010110000000100100000000100  bit map

Fields present in the above variable length message record:
2-7-12-28-32-39-41-42-50-53-62
Data elements

Data elements are the individual fields carrying the transaction information. There are up to 128 data elements specified in the original AS2805 standard, and up to 192 data elements in later releases.

While each data element has a specified meaning and format, the standard also includes some general purpose data elements and system- or country-specific data elements which vary enormously in use and form from implementation to implementation.

Each data element is described in a standard format which defines the permitted content of the field (numeric, binary, etc.) and the field length (variable or fixed), according to the following table:
Abbreviation 	Meaning
a 	Alpha, including blanks
n 	Numeric values only
s 	Special characters only
an 	Alphanumeric
as 	Alpha & special characters only
ns 	Numeric and special characters only
ans 	Alphabetic, numeric and special characters.
b 	Binary data
z 	Tracks 2 and 3 code set as defined in ISO/IEC 7813 and ISO/IEC 4909 respectively
. or .. or … 	variable field length indicator, each . indicating a digit.
x or xx or xxx 	fixed length of field or maximum length in the case of variable length fields.

Additionally, each field may be either fixed or variable length. If variable, the length of the field will be preceded by a length indicator.
Type 	Meaning
Fixed 	no field length used
LLVAR or (..xx) 	Where LL < 100, means two leading digits LL specify the field length of field VAR
LLLVAR or (…xxx) 	Where LLL < 1000, means three leading digits LLL specify the field length of field VAR
LL and LLL are hex or ASCII. A VAR field can be compressed or ASCII depending of the data element type. 	LL can be 1 or 2 bytes. For example, if compressed as one hex byte, ’27x means there are 27 VAR bytes to follow. If ASCII, the two bytes ’32x, ’37x mean there are 27 bytes to follow. 3 digit field length LLL uses 2 bytes with a leading ‘0’ nibble if compressed, or 3 bytes if ASCII. The format of a VAR data element depends on the data element type. If numeric it will be compressed, e.g. 87456 will be represented by 3 hex bytes ‘087456x. If ASCII then one byte for each digit or character is used, e.g. ’38x, ’37x, ’34x, ’35x, ’36x.
AS2805-defined data elements Data element 	Type 	Usage
1 	b 64 	Bit map (b 128 if secondary is present and b 192 if tertiary is present)
2 	n ..19 	Primary account number (PAN)
3 	n 6 	Processing code
4 	n 12 	Amount, transaction
5 	n 12 	Amount, settlement
6 	n 12 	Amount, cardholder billing
7 	n 10 	Transmission date & time
8 	n 8 	Amount, cardholder billing fee
9 	n 8 	Conversion rate, settlement
10 	n 8 	Conversion rate, cardholder billing
11 	n 6 	Systems trace audit number
12 	n 6 	Time, local transaction (hhmmss)
13 	n 4 	Date, local transaction (MMDD)
14 	n 4 	Date, expiration
15 	n 4 	Date, settlement
16 	n 4 	Date, conversion
17 	n 4 	Date, capture
18 	n 4 	Merchant type
19 	n 3 	Acquiring institution country code
20 	n 3 	PAN extended, country code
21 	n 3 	Forwarding institution. country code
22 	n 3 	Point of service entry mode
23 	n 3 	Application PAN number
24 	n 3 	Function code (ISO 8583:1993)/Network International identifier (NII)
25 	n 2 	Point of service condition code
26 	n 2 	Point of service capture code
27 	n 1 	Authorizing identification response length
28 	n 8 	Amount, transaction fee
29 	n 8 	Amount, settlement fee
30 	n 8 	Amount, transaction processing fee
31 	n 8 	Amount, settlement processing fee
32 	n ..11 	Acquiring institution identification code
33 	n ..11 	Forwarding institution identification code
34 	n ..28 	Primary account number, extended
35 	z ..37 	Track 2 data
36 	n …104 	Track 3 data
37 	an 12 	Retrieval reference number
38 	an 6 	Authorization identification response
39 	an 2 	Response code
40 	an 3 	Service restriction code
41 	ans 16 	Card acceptor terminal identification
42 	ans 15 	Card acceptor identification code
43 	ans 40 	Card acceptor name/location (1-23 address 24-36 city 37-38 state 39-40 country)
44 	an ..25 	Additional response data
45 	an ..76 	Track 1 data
46 	an …999 	Additional data – ISO
47 	an …999 	Additional data – national
48 	an …999 	Additional data – private
49 	an 3 	Currency code, transaction
50 	an 3 	Currency code, settlement
51 	an 3 	Currency code, cardholder billing
52 	b 64 	Personal identification number data
53 	n 18 	Security related control information
54 	an …120 	Additional amounts
55 	ans …999 	Reserved ISO
56 	ans …999 	Reserved ISO
57 	ans …999 	Reserved national
58 	ans …999 	Reserved national
59 	ans …999 	Reserved for national use
60 	an .7 	Advice/reason code (private reserved)
61 	ans …999 	Reserved private
62 	ans …999 	Reserved private
63 	ans …999 	Reserved private
64 	b 16 	Message authentication code (MAC)
65 	b 64 	*Bit indicator of tertiary bitmap only*, tertiary bitmap data follows secondary in message stream.
66 	n 1 	Settlement code
67 	n 2 	Extended payment code
68 	n 3 	Receiving institution country code
69 	n 3 	Settlement institution country code
70 	n 3 	Network management information code
71 	n 4 	Message number
72 	ans …999 	Data record (ISO 8583:1993)/n 4 Message number, last(?)
73 	n 6 	Date, action
74 	n 10 	Credits, number
75 	n 10 	Credits, reversal number
76 	n 10 	Debits, number
77 	n 10 	Debits, reversal number
78 	n 10 	Transfer number
79 	n 10 	Transfer, reversal number
80 	n 10 	Inquiries number
81 	n 10 	Authorizations, number
82 	n 12 	Credits, processing fee amount
83 	n 12 	Credits, transaction fee amount
84 	n 12 	Debits, processing fee amount
85 	n 12 	Debits, transaction fee amount
86 	n 15 	Credits, amount
87 	n 15 	Credits, reversal amount
88 	n 15 	Debits, amount
89 	n 15 	Debits, reversal amount
90 	n 42 	Original data elements
91 	an 1 	File update code
92 	n 2 	File security code
93 	n 5 	Response indicator
94 	an 7 	Service indicator
95 	an 42 	Replacement amounts
96 	an 8 	Message security code
97 	n 16 	Amount, net settlement
98 	ans 25 	Payee
99 	n ..11 	Settlement institution identification code
100 	n ..11 	Receiving institution identification code
101 	ans 17 	File name
102 	ans ..28 	Account identification 1
103 	ans ..28 	Account identification 2
104 	ans …100 	Transaction description
105 	ans …999 	Reserved for ISO use
106 	ans …999 	Reserved for ISO use
107 	ans …999 	Reserved for ISO use
108 	ans …999 	Reserved for ISO use
109 	ans …999 	Reserved for ISO use
110 	ans …999 	Reserved for ISO use
111 	ans …999 	Reserved for ISO use
112 	ans …999 	Reserved for national use
113 	n ..11 	Authorizing agent institution id code
114 	ans …999 	Reserved for national use
115 	ans …999 	Reserved for national use
116 	ans …999 	Reserved for national use
117 	ans …999 	Reserved for national use
118 	ans …999 	Reserved for national use
119 	ans …999 	Reserved for national use
120 	ans …999 	Reserved for private use
121 	ans …999 	Reserved for private use
122 	ans …999 	Reserved for private use
123 	ans …999 	Reserved for private use
124 	ans …255 	Info text
125 	ans ..50 	Network management information
126 	ans …999 	Issuer trace id
127 	ans …999 	Reserved for private use
128 	b 16 	Message authentication code

Implementation and understanding the Code

The first step to understanding the packing and unpacking of the message fields would be to create a class that describes then entire structure.

First we create a class and create a dictionary that describes the message formats:

 #2805 contants
 _DEF = {}
 # Every _DEF has:
 # _DEF[N] = [X, Y, Z, W, K]
 # N = bitnumber
 # X = smallStr representation of the bit meanning
 # Y = large str representation
 # Z = length indicator of the bit (F, LL, LLL, LLLL, LLLLL, LLLLLL)
 # W = size of the information that N need to has
 # K = type os values a, an, ans, n, xn, b
 _DEF[1] = ['BM', 'Bit Map Extended', 'F', 8, 'b']
 _DEF[2] = ['2', 'Primary Account Number (PAN)', 'LL', 19, 'n']
 _DEF[3] = ['3', 'Processing Code', 'F', 6, 'n']
 _DEF[4] = ['4', 'Amount Transaction', 'F', 12, 'n']
 _DEF[5] = ['5', 'Amount Settlement', 'F', 12, 'n']
 _DEF[7] = ['7', 'Transmission Date and Time', 'F', 10, 'n']
 _DEF[9] = ['9', 'Conversion Rate, Settlement', 'F', 8, 'n']
 _DEF[10] = ['10', 'Conversion Rate, Cardholder Billing', 'F', 8, 'n']
 _DEF[11] = ['11', 'Systems Trace Audit Number', 'F', 6, 'n']
 _DEF[12] = ['12', 'Time, Local Transaction', 'F', 6, 'n']
 _DEF[13] = ['13', 'Date, Local Transaction', 'F', 4, 'n']
 _DEF[14] = ['14', 'Date, Expiration', 'F', 4, 'n']
 _DEF[15] = ['15', 'Date, Settlement', 'F', 4, 'n']
 _DEF[16] = ['16', 'Date, Conversion', 'F', 4, 'n']
 _DEF[18] = ['18', 'Merchant Type', 'F', 4, 'n']
 _DEF[22] = ['22', 'POS Entry Mode', 'F', 3, 'n']
 _DEF[23] = ['23', 'Card Sequence Number', 'F', 3, 'n']
 _DEF[25] = ['25', 'POS Condition Code', 'F', 2, 'n']
 _DEF[28] = ['28', 'Amount, Transaction Fee', 'F', 9, 'xn']
 _DEF[32] = ['32', 'Acquiring Institution ID Code', 'LL', 11, 'n']
 _DEF[33] = ['33', 'Forwarding Institution ID Code', 'LL', 11, 'n']
 _DEF[35] = ['35', 'Track 2 Data', 'LL', 37, 'an']
 _DEF[37] = ['37', 'Retrieval Reference Number', 'F', 12, 'an']
 _DEF[38] = ['38', 'Authorization ID Response', 'F', 6, 'an']
 _DEF[39] = ['39', 'Response Code', 'F', 2, 'an']
 _DEF[41] = ['41', 'Card Acceptor Terminal ID', 'F', 8, 'ans']
 _DEF[42] = ['42', 'Card Acceptor ID Code', 'F', 15, 'ans']
 _DEF[43] = ['43', 'Card Acceptor Name Location', 'F', 40, 'asn']
 _DEF[44] = ['44', 'Additional Response Data', 'LL', 25, 'ans']
 _DEF[47] = ['47', 'Additional Data National', 'LLL', 999, 'ans']
 _DEF[48] = ['48', 'Additional Data Private', 'LLL', 999, 'ans']
 _DEF[49] = ['49', 'Currency Code, Transaction', 'F', 3, 'n']
 _DEF[50] = ['50', 'Currency Code, Settlement', 'F', 3, 'n']
 _DEF[51] = ['51', 'Currency Code, Billing', 'F', 3, 'n']
 _DEF[52] = ['52', 'PIN Data', 'F', 8, 'b']
 _DEF[53] = ['53', 'Security Related Control Information', 'F', 48, 'b']
 _DEF[55] = ['55', 'ICC Data', 'LLL', 999, 'b']
 _DEF[57] = ['57', 'Amount Cash', 'F', 12, 'n']
 _DEF[58] = ['58', 'Ledger Balance', 'F', 12, 'n']
 _DEF[59] = ['59', 'Account Balance', 'F', 12, 'n']
 _DEF[64] = ['64', 'Message Authentication Code', 'F', 8, 'b']
 _DEF[66] = ['66', 'Settlement Code', 'F', 1, 'n']
 _DEF[70] = ['70', 'Network Management Information Code', 'F', 3, 'n']
 _DEF[74] = ['74', 'Credits, Number', 'F', 10, 'n']
 _DEF[75] = ['75', 'Credits, Reversal Number', 'F', 10, 'n']
 _DEF[76] = ['76', 'Debits, Number', 'F', 10, 'n']
 _DEF[77] = ['77', 'Debits, Reversal Number', 'F', 10, 'n']
 _DEF[78] = ['78', 'Transfer, Number', 'F', 10, 'n']
 _DEF[79] = ['79', 'Transfer, Reversal Number', 'F', 10, 'n']
 _DEF[80] = ['80', 'Inquiries, Number', 'F', 10, 'n']
 _DEF[81] = ['81', 'Authorizations, Number', 'F', 10, 'n']
 _DEF[83] = ['83', 'Credits, Transaction Fee Amount', 'F', 12, 'n']
 _DEF[85] = ['85', 'Debits, Transaction Fee Amount', 'F', 12, 'n']
 _DEF[86] = ['86', 'Credits, Amount', 'F', 16, 'n']
 _DEF[87] = ['87', 'Credits, Reversal Amount', 'F', 16, 'n']
 _DEF[88] = ['88', 'Debits, Amount', 'F', 16, 'n']
 _DEF[89] = ['89', 'Debits, Reversal Amount', 'F', 16, 'n']
 _DEF[90] = ['90', 'Original Data Elements', 'F', 42, 'n']
 _DEF[97] = ['97', 'Amount, Net Settlement', 'F', 17, 'xn']
 _DEF[99] = ['99', 'Settlement Institution ID Code', 'LL', 11, 'n']
 _DEF[100] = ['100', 'Receiving Institution ID Code', 'LL', 11, 'n']
 _DEF[112] = ['112', 'Key Management Data', 'LLL', 999, 'b']
 _DEF[118] = ['118', 'Cash Total Number', 'LLL', 10, 'n']
 _DEF[119] = ['119', 'Cash Total Amount', 'LLL', 10, 'n']
 _DEF[128] = ['128', 'MAC Extended', 'F', 8, 'b']

To write this structure should not be that difficult and should come directly out of the specification provided my your institution.

As for the infamous bitmaps we are going to describe a structure for them as well, with an array to track the bit positions.

 #Attributes
 # Bits to be set 00000000 -> _BIT_POSITION_1 ... _BIT_POSITION_8
 _BIT_POSITION_1 = 128 # 10 00 00 00
 _BIT_POSITION_2 = 64 # 01 00 00 00
 _BIT_POSITION_3 = 32 # 00 10 00 00
 _BIT_POSITION_4 = 16 # 00 01 00 00
 _BIT_POSITION_5 = 8 # 00 00 10 00
 _BIT_POSITION_6 = 4 # 00 00 01 00
 _BIT_POSITION_7 = 2 # 00 00 00 10
 _BIT_POSITION_8 = 1 # 00 00 00 01

 #Array to translate bit to position
 _TMP = [0, _BIT_POSITION_8, _BIT_POSITION_1, _BIT_POSITION_2, _BIT_POSITION_3, _BIT_POSITION_4, _BIT_POSITION_5,
 _BIT_POSITION_6, _BIT_POSITION_7]
 _EMPTY_VALUE = 0

Now we simply add a heap of helper functions from  http://www.vulcanno.com.br/python/ISO8583.html to fill and decode the data structure.

I have created a client and a server as part of the source code, so you can test your own implementation of the AS2805 protocol

Simple as pie!
