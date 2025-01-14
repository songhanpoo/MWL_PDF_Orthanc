#### https://www.markdownguide.org/cheat-sheet/
## TO BUILD:

1.  Clone the directory.  The default setup is for "SslEnabled" : false and "DicomTlsEnabled" : false so you don't have to mess too much with certificates, but you still need to do steps 4-6.
2.  Copy sample.env to .env and configure.  The defaults are probably OK, but if you want to receive e-mails using that feature you'll need to add your own maileserver credentials:
    PYTHON_EMAIL_HOST, PYTHON_EMAIL_USER, PYTHON_EMAIL_PASS, PYTHON_EMAIL_FRO, PYTHON_EMAIL_TO
3.  Copy sample.env_pacs to .env_pacs and configure.
4.  Use 'sh generate-tls.sh' file in the tls folder to create some server-crt.pem and server-key.pem files for SSL if you want to use that.
5.  Copy the generated crt and key to orthanc.crt and orthanc.key in the tls_dicom folder to enable DICOM TLS.
6.  Combine those in combined.crt.pem in the tls_dicom folder to enable SSL for the REST API and Explorer.
7.  There might be a way to enable the CA files as well so that the certificate verifies, or just use your own.
8.  Change to the root of the folder and execute:  sudo docker-compose up --build (-d optionally).
9.  Verify no errors.

If you are building from scratch the "RIS_DB", orthanc_ris should be created and populated automatically.
Otherwise, see mysql_init for the init script.

The default setup is to store generated MWL's in the DB, not the File system and to also generate responses using the DB.

#### Orthanc is accessible on:

https://localhost:8042 or http://localhost:8042 depending upon if you setup SSL connections.

#### PHPMyAdmin is accessible on:

http://localhost:11080
user: root, password:  root

#### PGAdmin is accessible on:

http://localhost:5050
user: sscotti@sias.dev, password:  postgres

#### Explorer2:

https://localhost:8042/ui/app/#/ or http://localhost:8042/ui/app/#/

#### Legacy Explorer

https://localhost:8042/app/explorer.html or http://localhost:8042/app/explorer.html

## Note about MedDream and OHIF viewers:

There are docker builds for both MedDream and OHIF included with the package, and they are for the most part integrated into the Explorer2 interface, although you made need to tweak the configuration a little, particularly for MedDream.  I would recommend getting a trial license if you really want to play around with that.  You might to play around with the certificates in the tls folder and in the nginx.conf files to get that all setup the way you want.

## Note about the orthanc_ris DB for MySQL and Postgres:

1.  This is still a work in progress, but note the following.
2.  The **mml** table stores data concerning MWL files that are created via the API, including the JSON and Binary DICOM format.  Responses can be generated from the DB vs. the native file system method as part of the core of Orthanc.
3.  **n_create** and **n_set** are not implemented, just 'placeholders' for now.
4.  **study_complete** is to store some data about when a study is marked complete via the API, for instances where the RIS or EMR provide a method to do that since Orthanc mostly relies on the study being Stable for a period of time.
5.  **study_first_instances** stores some data about the study when the NEW_STUDY event is triggered.

#### You can just upload a new study via the API (Explorer 2) to see how the study_first_instances works (MySQL only for now).

#### The study_complete is just an API / curl call like:

`
     curl -k -v http://localhost:8042/studies/study_complete_DB -d '{"Tech": "1:SDS", "ID": "d8cadbbc-ad5e1f78-d6361373-86f22b9f-8c0100e2"}'
`


#### To test creating a MWL from JSON use:


curl -k -X POST  http://localhost:8042/mwl/create_from_json -d '
{
"AccessionNumber": "CMACC00000002",
"AdditionalPatientHistory": "test",
"AdmittingDiagnosesDescription": "",
"Allergies": "",
"ImageComments": "Tech:  TEST",
"MedicalAlerts": "TEST",
"Modality": "MR",
"Occupation": "",
"OperatorsName": "Tech^SDS",
"PatientAddress": "^^Vienna^OS^1160^AT",
"PatientBirthDate": "20010101",
"PatientComments": "",
"PatientID": "PatientID",
"PatientName": "Person^Test^",
"PatientSex": "M",
"PatientSize": "",
"PatientTelecomInformation": "^^^",
"PatientWeight": "",
"ReferringPhysicianIdentificationSequence": [{
"InstitutionName": "InstitutionName",
"PersonIdentificationCodeSequence": [{
"CodeMeaning": "Local Code",
"CodeValue": "0001",
"CodingSchemeDesignator": "L"
}],
"PersonTelephoneNumbers": "^PN^^"
}],
"ReferringPhysicianName": "0001:Scotti^Stephen^Douglas^Dr.",
"ScheduledProcedureStepSequence": [{
"Modality": "MR",
"ScheduledProcedureStepDescription": "MRI BRAIN / BRAIN STEM - WITHOUT CONTRAST",
"ScheduledProcedureStepID": "0001",
"ScheduledProcedureStepStartDate": "20210704",
"ScheduledProcedureStepStartTime": "110000",
"ScheduledProtocolCodeSequence": [{
"CodeMeaning": "",
"CodeValue": "70551",
"CodingSchemeDesignator": "C4"
}],
"ScheduledStationAETitle": "NmrEsaote"
}],
"SpecificCharacterSet": "ISO_IR 192",
"StudyInstanceUID": "StudyInstanceUID"
}
'


#### To test an MWL query using findscu use the following.

####   Add +tla -ic  if using Dicom TLS

findscu  localhost 4242 -W -v -d -k "AccessionNumber" \
-k "Modality" \
-k "InstitutionName" \
-k "ReferringPhysicianName" \
-k "ReferencedStudySequence[0]" \
-k "ReferencedPatientSequence[0]" \
-k "PatientName" \
-k "PatientID" \
-k "PatientBirthDate" \
-k "PatientSex" \
-k "PatientAge" \
-k "PatientWeight" \
-k "MedicalAlerts" \
-k "Allergies" \
-k "PregnancyStatus" \
-k "StudyInstanceUID" \
-k "StudyID" \
-k "RequestingPhysician" \
-k "RequestedProcedureDescription" \
-k "RequestedProcedureCodeSequence[0]" \
-k "AdmissionID" \
-k "SpecialNeeds" \
-k "CurrentPatientLocation" \
-k "PatientState" \
-k "ScheduledProcedureStepSequence[0]" \
-k "RequestedProcedureID" \
-k "RequestedProcedurePriority" \
-k "PatientTransportArrangements" \
-k "ConfidentialityConstraintOnPatientDataDescription"


####   To Store a PDF using BASE64 or HTML: Replace the studyuuid with the one that you want to operate on and use the commands below:

Hello World base64 PDF:  

curl -k http://localhost:8042/pdfkit/htmltopdf -d  '{
"method":"base64",
"title":"BASE64 TO PDF",
"author": "Stephen D. Scotti",
"studyuuid":"d8cadbbc-ad5e1f78-d6361373-86f22b9f-8c0100e2",
"base64":"JVBERi0xLjcKCjEgMCBvYmogICUgZW50cnkgcG9pbnQKPDwKICAvVHlwZSAvQ2F0YWxvZwogIC9QYWdlcyAyIDAgUgo+PgplbmRvYmoKCjIgMCBvYmoKPDwKICAvVHlwZSAvUGFnZXMKICAvTWVkaWFCb3ggWyAwIDAgMjAwIDIwMCBdCiAgL0NvdW50IDEKICAvS2lkcyBbIDMgMCBSIF0KPj4KZW5kb2JqCgozIDAgb2JqCjw8CiAgL1R5cGUgL1BhZ2UKICAvUGFyZW50IDIgMCBSCiAgL1Jlc291cmNlcyA8PAogICAgL0ZvbnQgPDwKICAgICAgL0YxIDQgMCBSIAogICAgPj4KICA+PgogIC9Db250ZW50cyA1IDAgUgo+PgplbmRvYmoKCjQgMCBvYmoKPDwKICAvVHlwZSAvRm9udAogIC9TdWJ0eXBlIC9UeXBlMQogIC9CYXNlRm9udCAvVGltZXMtUm9tYW4KPj4KZW5kb2JqCgo1IDAgb2JqICAlIHBhZ2UgY29udGVudAo8PAogIC9MZW5ndGggNDQKPj4Kc3RyZWFtCkJUCjcwIDUwIFRECi9GMSAxMiBUZgooSGVsbG8sIHdvcmxkISkgVGoKRVQKZW5kc3RyZWFtCmVuZG9iagoKeHJlZgowIDYKMDAwMDAwMDAwMCA2NTUzNSBmIAowMDAwMDAwMDEwIDAwMDAwIG4gCjAwMDAwMDAwNzkgMDAwMDAgbiAKMDAwMDAwMDE3MyAwMDAwMCBuIAowMDAwMDAwMzAxIDAwMDAwIG4gCjAwMDAwMDAzODAgMDAwMDAgbiAKdHJhaWxlcgo8PAogIC9TaXplIDYKICAvUm9vdCAxIDAgUgo+PgpzdGFydHhyZWYKNDkyCiUlRU9G","return":1,"attach":1
}'

curl -k http://localhost:8042/pdfkit/htmltopdf -d '{
"method":"html","title":"HTML To PDF",
"author": "Stephen D. Scotti",
"studyuuid":"d8cadbbc-ad5e1f78-d6361373-86f22b9f-8c0100e2",
"html":"Basically put any valid HTML, including CSS that can be rendered by wkhtmltopdf",
"return":1,
"attach":1
}'

#### To test Key Image, replace the StudyInstanceUID with one on your server.  If there is no KEY_IMAGE series it will create one.  If there already is one it will add an instance to the series.

There does seem to be an issue with running this in the Python Plug-in.  I've implemented the same from PHP via an API and that works better as there are no 'hangs'.
I've experienced Orthanc hanging sometimes in some of my setups when the study for which a key image is being made is concurrently being modified by Orthanc, or when Orthanc is conccurrently
receiving studies from a modality via the DICOM protocol.

curl -k http://localhost:8042/make_key_image -d '{
"StudyInstanceUID": "1.3.6.1.4.1.5962.1.2.1.20040119072730.12322",
"ImageComments": "TEST",
"data_url" : "data:image/jpeg;base64,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AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAH/2Q=="
}'

#### Custom Archive called as usual:

curl -k -v http://localhost:8042/studies/orthanc_uuid/archive > Study.zip

The Python Plug-in is configured to create a custom archive having the Radiant Viewer and the archive in a single .zip file