# Kasten firestore 

Include the backup of your firestore database with your Kasten backup 

## How it works 

The usual way as described in the tutorial [art of blueprint](https://www.kasten.io/kubernetes/resources/how-to-guides/blueprints) : 

A secrets containing the google service account key file in charge of making the backup will be annotated with the blueprint 
[firestore-bp]((./firstore-bp.yaml)). When Kasten backup, restore the namespace or delete the backup Kasten will detect
this annotation and the corresponding backup, restore or delete action of the blueprint will be called, with the secret as the context 
of the backup.

## Prerequisite 

### Create a service account to operate backup and restore 

You need to create a google service account that can operate backup and export to your google bucket. 

Change those variables according to your setting 

```
PROJECT_ID=k8shardwaysysmic
FIRESTORE_SA=firestore-manager
BUCKET=firestore-backup-mcourcy
DATABASE="(default)"
BUCKET_PREFIX=backup/firestore/$DATABASE
```

```
gcloud config set project $PROJECT_ID
gcloud iam service-accounts create $FIRESTORE_SA
gcloud iam service-accounts keys create --iam-account=$FIRESTORE_SA@$PROJECT_ID.iam.gserviceaccount.com $FIRESTORE_SA-sa-key.json
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:$FIRESTORE_SA@$PROJECT_ID.iam.gserviceaccount.com --role roles/datastore.owner

# Create a bucket role and limit the sa with this role to this specific bucket 
gcloud iam roles create kasten_bucket --project=$PROJECT_ID --file=kasten-bucket.yaml
gsutil iam ch serviceAccount:$FIRESTORE_SA@$PROJECT_ID.iam.gserviceaccount.com:projects/$PROJECT_ID/roles/kasten_bucket gs://$BUCKET
```

### Test this service account can proceed the backup of firestore 

```
gcloud auth activate-service-account --key-file=$FIRESTORE_SA-sa-key.json
gcloud firestore export --database=$DATABASE gs://$BUCKET/$BUCKET_PREFIX/`date +%m-%d-%Y`
gsutil rm -r gs://$BUCKET/$BUCKET_PREFIX/`date +%m-%d-%Y`
```

you should get an output similar to this one 
```
Waiting for [projects/k8shardwaysysmic/databases/(default)/operations/ASA1MDEwODA1MzUJGnRsdWFmZWQHEjN3LXVlLXNib2otbmltZGEQCigS] to finish...done.                                              
metadata:
  '@type': type.googleapis.com/google.firestore.admin.v1.ExportDocumentsMetadata
  operationState: PROCESSING
  outputUriPrefix: gs://firestore-backup-mcourcy/backup/firestore/(default)/07-28-2023
  startTime: '2023-07-28T15:32:06.251480Z'
name: projects/k8shardwaysysmic/databases/(default)/operations/ASA1MDEwODA1MzUJGnRsdWFmZWQHEjN3LXVlLXNib2otbmltZGEQCigS
```

## Deploy 

Create the blueprint 
```
kubectl create -f firestore-bp.yaml
```

Create a secret with the info 
```
kubectl create secret generic firestore \
  --from-file=key-file=$FIRESTORE_SA-sa-key.json \
  --from-literal=project=$PROJECT_ID \
  --from-literal=bucket=$BUCKET \
  --from-literal=database=$DATABASE
```

Annotate the secret 
```
kubectl annotate secret firestore kanister.kasten.io/blueprint=firestore-bp
```

## Backup and restore 

Just backup or restore the namespace with kasten it will backup or restore the database at the same time of the kasten restorepoint.

## Debug 

You can use this command to outpput the kubetask and troubleshoot 
```
while true; do kubectl logs -f -l createdBy=kanister; sleep 2; done
```
