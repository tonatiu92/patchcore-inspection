# PseudoCode Explanation

SHEMA ON MIRO https://miro.com/welcomeonboard/TTZ4cXdjMlNQaFFuWEFaTzhBUkJub0NjRlhSUFVuTHVVMUVNeUJmQkFZYWtXZFhGcnVwSHNhOWxlNHFCSmZOSXwzNDU4NzY0NTMxMzQ4MjMwMTgyfDI=?share_link_id=395795269944

## Command Description

### COMPUTER PARAMETERS
* --gpu 0 
* --seed 0 
* --save_patchcore_model \
* --log_group IM224_WR50_L2-3_P01_D1024-1024_PS-3_AN-1_S0 
* --log_online 
* --log_project MVTecAD_Results results \

### PATCHCORE PARAMETERS
* patch_core (Parameters for the function patchcore line 259
	* -b wideresnet50  				//define the backbone_names list
	* -le layer2 -le layer3 			//define the layers to extract
	* --faiss_on_gpu \
	* --pretrain_embed_dimension 1024  
	* --target_embed_dimension 1024 
	* --anomaly_scorer_num_nn 1 
	* --patchsize 3 \

### SAMPLER PARAMETERS
* sampler -p 0.1 approx_greedy_coreset  ##sampling method with percentage in this case 10%

### DATASETS PARAMETERS
* dataset 
    * --resize 256 
    * --imagesize 224 
    * "${dataset_flags[@]}" mvtec $datapath

* datapath=/path_to_mvtec_folder/mvtec datasets=('bottle' 'cable' 'capsule' 'carpet' 'grid' 'hazelnut'
'leather' 'metal_nut' 'pill' 'screw' 'tile' 'toothbrush' 'transistor' 'wood' 'zipper')
* dataset_flags=($(for dataset in "${datasets[@]}"; do echo '-d '$datasetdone))

## run_pathcore()

### 2.A.1 
Defining imagesize using get_dataloaders(seed): line 357

the fuction is called from methods["get_dataloaders"] line 51
- **get_dataloaders(seed):**
	- extracting all dataset from the list datapath according the parameters. In the implementation they called the datasets "mvtec" from library "patchcore.datasets.mvtec", "MVTecDataset" (dataset_info in the code). it splits the dataset mvtec in train/test using the dataset_library.DatasetSplit.TRAIN and .TEST from patchcore/datasets/mvtec.py
	- torch.utils.data.DataLoader: the DataLoader class in PyTorch that helps us to load and iterate over elements in a dataset. it is a pipeline to process the train and test data 
		
	- This function return a list, for each dataset, of dictionary containing the training, validation and testing data_loaders. 
		
	- Then we will able to iterate over each dataloaders as see in line 65
		
- **The imagesize** is the imagesize of the dataloader["training].dataset.imagesize also definedby command --imagesize 224



***BEGINNING THE LOOP OVER EACH DATALOADERS (OR OVER EACH DATASET STRUCTURES)***

### 2.A.2
- Defining sampling method using **get_sampler(device)** line 322 the function is called from methods["get_sampler"]: line 81

	- it only looks at the name defined in the command (in our case **approx_greedy_coreset** with percentage of 10%)
	- it returns the instance of the class from patchcoere.sampler library (in our case **patchcore.sampler.ApproximateGreedyCoreSampler(percentage,device)**)


### 2.A.3

***DEFINING ALL THE PATCHCORE IN THE DATALOADER, We are getting a patchcorelist line 84***
- function called by method["get_patchcore"](imagesize, sampler, device) ///line(84)
- **get_patchcore(input_shape:   ,sampler: SamplerMethod, device: Torch)**:
		
	- line 275 to 282 (not applying in our case because we have only one backbone_name:
        - it aggregate each layer to a backbone/backbone are the architecture to encode
			in our case layers_to_extract_from_coll is a single list with the layers to extract from (layer 2 and 3)
		
	- for backbone_name, layers_to_extract_from in zip(backbone_names, layers_to_extract_from_coll):
		- it eval the backbone method with **patchcores.backbones.load(backbone_name)**
		- it define the nn_method **patchcore.common.FaissNN(faiss_on_gpu, faiss_num_workers)** params defined in the command line in our case (True, 4(the default value))
		- creates the instance of **class patchcore.patchcore.PatchcCore(device)**
		- **load the instance created patchcore_instance.load(...)**
			- this function define the main paramaeters of the patchcore defining attributes or creating the attributes instance
			1. backbone using **backbone.to(device)** .to is a torch function that return a Tensor with a specified dtype
            2. **layers_to_extract** (2 and 3 in our case)
            3. **input_shape** the imagesize in our case of 224
            4. device
            5. instance **patch_maker** using **PatchMakerClass (patchsize 3 and stride 1 in our case)**
            6. instance **forward_modules** using **torch.nn.ModuleDict({})** that Holds submodules in a dictionary. The module of this instance are:
                * **"feature_aggregator"**: instance feature_aggregator from **patchcore.common.NetworkFeatureAggregator( self.backbone, self.layers_to_extract_from, self.device)** that Runs a network only to the last layer of the list of layers where network features should be extracted from.
				* **"preprocessing"**: instance preprocessing from **patchcore.common.Preprocessing(feature_dimension, pretrain_embed_dimension** feature_dimension is defined in  "feature_aggregator" //it is the input_dim of preproc pretrain_embed_dimension 1024 in our case (see commands) // it is the output_dim of preproc
				* **"preadapt_aggregator"**: instance of **patchcore.common.Aggregator(target_embed_dimension)** in our case 1024, Aggregator Returns reshaped and average pooled features
            7. instance **anomaly_scorer** from **patchcore.common.NearestNeighbourScorer(n_nearest_neighbours=anomaly_score_num_nn, nn_method=nn_method**) in our case anomaly_score_num_nn = 1  and nn_method = ii)patchcore.common.FaissNN(faiss_on_gpu, faiss_num_workers this instance is the Neearest-Neighbourhood Anomaly Scorer class.
            8. instance **anomaly_segmentor** from **patchcore.common.RescaleSgementor(device, target_size=input_shape[-2:])** that convert to segmentation 
            9. **feature_sampler**, the sampler defined in **2.A.2**
			10. return the loaded patchcore in our case it will be a list of 1 patchcore
				
***FOR EACH PATHCORE IN THE LIST OF PATCHCORE***
			
### 2.A.4 FITTING THE PATHCORE IN THE TRAINING DATALOADER
- We call the **PatchCore.fit(dataloader["training"])** line 97
- This function call in pathcore.patchcore file the **fill_memory_bank(training_data)** line 153 of file patchcore.patchcore
	- for each image in input_data (training_data) if the image is a dict the image treated is image["image"] (in our case i don't know, not very important)
	- append to the list of features with **image_to_feature(image)**
        1. using **torch.no_grad()** that set all requires_grad to false for each tensor. 
		2. it set each input_image as Tensor of type device
		3. this last function called the patchcore.patchcore function **_embed(input_image)** (here call without provide patch shapes)
			* call the attribute **self.forward_modules["feature_aggregator"](images)**, this function will apply the backbone to the images
			* **extract all features for each layer**
			* **patchify** each features **self.pathchMaker.patchify(patchsize, return_spatial_info = True (in our case))**
                - Convert a tensor into a tensor of respective patches.
										Args:
											x: [torch.Tensor, bs x c x w x h]
										Returns:
											x: [torch.Tensor, bs * w//stride * h//stride, c, patchsize, patchsize]
			* for each feature make the **bi-linear interpolation**
			* Apply the **forward_modules["prepocessing"]** on features
			* Apply the **forward_modules["preadapt_aggregator"]** on features
							return the detach(features) because we don't want the computational graph, we want a np.ndarray
	- **concatenate** the features
	- apply the **sampling** on features
	- fit the **anomaly_scorer.fit(detection_features=[features])** defined in 7. from **2.A.3** it applies the nn_methods (the **FaissNN** in our case) with **nn_method.fit(features)** that Adds features to the FAISS search index
						
			
***FOR EACH PATHCORE IN THE LIST OF PATCHCORE***

### 2.B.1
- We call the PatchCore.predict(dataloader["testing"]) line 10
- This function call the patchcore.patchcore.precdict(data_loader["testing"])
	- if instance of data_loader (in this case yes)
						call _predict_dataloader(data) that provides anomaly_scores/maps for full dataloader
	- for each image in dataloader
		- _score, _mask = self._predict(image) where patch_scores is responsible for segmentation and images_score for anomaly score
			1. use batchsize = images.shape[0] #number of training example
			2. call **_embed(input_images, provide_patch_shapes=True)** as **2.A.4**  with provide patch shape 
			3. **patch_scores** = image_scores = **self.anomaly_scorer.predict()** that  Searches for nearest neighbours of test images in all support training images.
			4. **image_scores** = **self.patch_maker.unpatch_score(image_scores, batchsize)** that only reshape the image_score according batchsize
			5. image_scores = **self.patch_maker.score**
			6. patch_scores = **self.patch_maker.unpatch_scores(patch_scores, batchsize=batchsize)**
			7. scales = patch_shapes[0]
			8. patch_scores = patch_scores.reshape(batchsize, scales[0], scales[1])
			9. **masks** = **self.anomaly_segmentor.convert_to_segmentation(patch_scores)**				
			10. return [score for score in image_scores], [mask for mask in masks]
		- return [score for score in image_scores], [mask for mask in masks]
	- return [score for score in image_scores], [mask for mask in masks]
### 2.B.2 
- append scores
### 2.B.3 
- append segmentation mask
			
### NORMALIZE the scores mean(scores-min/max-min)
### NORMALIZE Segmentation

***END DATALOADERS LOOP***		
### PART 3:
- AUC and ROC metrics save using patchcore.metrics