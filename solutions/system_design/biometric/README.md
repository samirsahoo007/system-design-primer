# Biometric System Architecture
Biometrics are computerized methods of identifying a person on the basis of physiological and observable qualities. Face, fingerprints, handwriting, iris, retinal and voice are the various characteristics which are measured in biometric techniques.

There are 2 phases of a Biometric system:

1. **Enrollment phase:**
In enrollment phase, biometric information of the user or person is recorded in a database. It is a one-time process, generally in this phase measurement of the appropriate information is done very precisely.
2. **Recognition phase:**
This is the second phase of the biometric system. This occurs when the detection part begins on the basis of the first phase for the authentication of the user. This phase must be quick, accurate and able to determine the authentication problem easily.

A **Biometric system architecture** has the following main components:

1. Sensor
2. Pre-processing
3. Feature extractor
4. Template generator
5. Matcher
6. Application device  

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/biometric.png)

## Sensor:
Sensor is the first block of the biometric system which collects all the important data for the biometrics. It is the interface between the system and the real world. Typically, it is an image acquisition system but it depends on the features or characteristics required that it has to replaced or not.

## Pre-processing:
It is the second block which executes all the pre-processing. Its function is to enhance the input and to eliminate the artifacts from the sensor, background noise, etc. It performs some kind of normalization.

## Feature extractor:
This is the third and the most important step in the biometric system. Extraction of features are to be done in order to identifying them at the later stage. The goal of a feature extractor is to characterize an object to be recognized by measurements.

## Template generator:
Template generator generates the templates that are used for the authentication with the help of the extracted features. A template is a vector of numbers or an image with distinct tracts. Characteristics obtained from the source groups together to form a template. Templates are being stored in the database for comparison and serves as a input for matcher.

## Matcher:
The matching phase is being performed by the use of a matcher. In this part, the procured template is given to a matcher that compares it with the stored templates using various algorithms such as Hamming distance, etc. After matching of the inputs, the results will be generated.

## Application device:
It is the device which uses the results of the biometric system. Iris recognition system and facial recognition system are some common examples of application devices.

