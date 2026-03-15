Meta Annotation Plot
====================

Upload metadata prediction results to visualize organ and phenotype distributions.

.. raw:: html

   <style>
     .upload-area { border: 2px dashed #ccc; border-radius: 12px; padding: 48px; text-align: center; margin-bottom: 24px; background: var(--color-background-secondary); cursor: pointer; transition: all 0.2s; }
     .upload-area:hover, .upload-area.dragover { border-color: #4a90d9; background: #f0f7ff; }
     .upload-area h3 { font-size: 18px; color: var(--color-foreground-secondary); margin-bottom: 8px; }
     .upload-area p { color: var(--color-foreground-muted); font-size: 13px; }
     .upload-area .format { background: var(--color-background-primary); border: 1px solid var(--color-background-border); border-radius: 6px; padding: 12px; margin-top: 16px; font-family: monospace; font-size: 12px; text-align: left; display: inline-block; }

     .tool-controls { display: none; gap: 16px; margin-bottom: 24px; flex-wrap: wrap; align-items: flex-end; }
     .tool-controls.visible { display: flex; }
     .tool-controls .control-group { display: flex; flex-direction: column; gap: 6px; }
     .tool-controls .control-group label { font-size: 13px; font-weight: 600; }
     .tool-controls select, .tool-controls button { padding: 10px 16px; border: 1px solid var(--color-background-border); border-radius: 8px; font-size: 14px; background: var(--color-background-primary); color: var(--color-foreground-primary); cursor: pointer; }
     .tool-controls button.primary { background: #4a90d9; color: #fff; border-color: #4a90d9; font-weight: 600; }
     .tool-controls button.primary:hover { background: #3a7bc8; }
     .tool-controls button.primary:disabled { background: #aaa; border-color: #aaa; cursor: not-allowed; }
     .tool-controls button.download { background: #28a745; color: #fff; border-color: #28a745; }

     .plot-container { background: var(--color-background-primary); border-radius: 12px; padding: 24px; box-shadow: 0 2px 8px rgba(0,0,0,0.08); margin-bottom: 24px; display: none; }
     .plot-container.visible { display: block; }
     .plot-container img { max-width: 100%; height: auto; display: block; margin: 0 auto; }

     .tool-log { background: #1e1e1e; border-radius: 8px; padding: 16px; max-height: 200px; overflow-y: auto; font-family: 'Consolas', 'Monaco', monospace; font-size: 13px; color: #ccc; line-height: 1.6; display: none; }
     .tool-log.visible { display: block; }
     .tool-log .info { color: #6cb6ff; }
     .tool-log .success { color: #7ee787; }
     .tool-log .error { color: #f85149; }
     .tool-log .stage { color: #d2a8ff; font-weight: bold; }

     .loading-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(255,255,255,0.9); display: none; flex-direction: column; align-items: center; justify-content: center; z-index: 1000; }
     .loading-overlay.visible { display: flex; }
     .spinner { width: 48px; height: 48px; border: 4px solid #eee; border-top-color: #4a90d9; border-radius: 50%; animation: tool-spin 0.8s linear infinite; }
     @keyframes tool-spin { to { transform: rotate(360deg); } }
     .loading-overlay p { margin-top: 16px; color: #666; font-size: 15px; }
   </style>

   <div class="loading-overlay" id="loading">
     <div class="spinner"></div>
     <p id="loading-text">Loading Pyodide...</p>
   </div>

   <div class="upload-area" id="upload-area">
     <h3>Drop CSV file here or click to upload</h3>
     <p>Expected columns (from meta.integrate() output):</p>
     <div class="format">Organ_pred, Organ_prob, Pheno_pred, Pheno_prob</div>
     <p style="margin-top:12px">Optionally include clinical metadata columns for grouped analysis</p>
     <input type="file" id="file-input" accept=".csv" style="display:none" />
   </div>

   <div class="tool-controls" id="controls">
     <div class="control-group">
       <label for="plot-type">Plot Type</label>
       <select id="plot-type">
         <option value="organ">Organ Distribution</option>
         <option value="pheno">Phenotype Distribution</option>
         <option value="grouped">Grouped Phenotype (by metadata)</option>
       </select>
     </div>
     <div class="control-group" id="group-col-group" style="display:none">
       <label for="group-col">Group By Column</label>
       <select id="group-col"></select>
     </div>
     <div class="control-group">
       <label>&nbsp;</label>
       <button class="primary" id="run-btn" onclick="generateMetaPlot()">Generate Plot</button>
     </div>
     <div class="control-group">
       <label>&nbsp;</label>
       <button class="download" id="download-btn" style="display:none" onclick="downloadMetaPlot()">Download PNG</button>
     </div>
   </div>

   <div class="plot-container" id="plot-container">
     <img id="plot-img" />
   </div>

   <div class="tool-log" id="log"></div>

   <script>
     let pyodide=null;
     function log(msg,cls='info'){const el=document.getElementById('log');el.innerHTML+=`<div class="${cls}">${msg}</div>`;el.scrollTop=el.scrollHeight;}
     function stage(msg){log(`[Stage] ${msg}`,'stage');}

     const uploadArea=document.getElementById('upload-area'),fileInput=document.getElementById('file-input');
     uploadArea.addEventListener('click',()=>fileInput.click());
     uploadArea.addEventListener('dragover',e=>{e.preventDefault();uploadArea.classList.add('dragover');});
     uploadArea.addEventListener('dragleave',()=>uploadArea.classList.remove('dragover'));
     uploadArea.addEventListener('drop',e=>{e.preventDefault();uploadArea.classList.remove('dragover');handleFile(e.dataTransfer.files[0]);});
     fileInput.addEventListener('change',e=>{if(e.target.files[0])handleFile(e.target.files[0]);});

     async function handleFile(file){
       if(!file.name.endsWith('.csv')){alert('Please upload a CSV file.');return;}
       document.getElementById('loading').classList.add('visible');
       document.getElementById('log').classList.add('visible');
       if(!pyodide){
         stage('Loading Pyodide engine...');
         const s=document.createElement('script');s.src='https://cdn.jsdelivr.net/pyodide/v0.26.3/full/pyodide.js';
         document.head.appendChild(s);await new Promise(r=>s.onload=r);
         pyodide=await loadPyodide();log('Pyodide loaded','success');
         stage('Installing packages...');document.getElementById('loading-text').textContent='Installing packages...';
         await pyodide.loadPackage(['numpy','pandas','matplotlib','micropip']);
         const mp=pyodide.pyimport('micropip');await mp.install('seaborn');log('Packages ready','success');
       }
       stage('Reading CSV...');
       pyodide.FS.writeFile('/data.csv',await file.text());
       const extra=(await pyodide.runPythonAsync(`
import pandas as pd
df=pd.read_csv('/data.csv')
base={'Organ_pred','Organ_prob','Pheno_pred','Pheno_prob'}
[c for c in df.columns if c not in base and not c.startswith('Unnamed')]
       `)).toJs();
       const gc=document.getElementById('group-col');gc.innerHTML='';
       for(const c of extra) gc.innerHTML+=`<option value="${c}">${c}</option>`;
       log(`Loaded: ${file.name} (${extra.length} extra columns)`,'success');
       uploadArea.style.display='none';document.getElementById('controls').classList.add('visible');
       document.getElementById('loading').classList.remove('visible');stage('Ready!');
     }

     document.getElementById('plot-type').addEventListener('change',()=>{
       document.getElementById('group-col-group').style.display=document.getElementById('plot-type').value==='grouped'?'flex':'none';
     });

     async function generateMetaPlot(){
       const btn=document.getElementById('run-btn');btn.disabled=true;btn.textContent='Generating...';stage('Generating plot...');
       const pt=document.getElementById('plot-type').value,gc=document.getElementById('group-col').value||'';
       try{
         await pyodide.runPythonAsync(`
import pandas as pd,numpy as np,matplotlib,io,base64
matplotlib.use('Agg')
import matplotlib.pyplot as plt;import seaborn as sns
df=pd.read_csv('/data.csv');plot_type='${pt}';group_col='${gc}'
if plot_type=='organ':
    fig,ax=plt.subplots(figsize=(8,5));counts=df['Organ_pred'].value_counts(normalize=True).sort_values()
    counts.plot.barh(ax=ax,color='skyblue',edgecolor='black',linewidth=1,width=0.8)
    ax.set_title('Predicted Organ Distribution',fontsize=14,fontweight='bold');ax.set_xlabel('Proportion');sns.despine()
elif plot_type=='pheno':
    fig,ax=plt.subplots(figsize=(8,5));counts=df['Pheno_pred'].value_counts(normalize=True).sort_values()
    counts.plot.barh(ax=ax,color='salmon',edgecolor='black',linewidth=1,width=0.8)
    ax.set_title('Predicted Phenotype Distribution',fontsize=14,fontweight='bold');ax.set_xlabel('Proportion');sns.despine()
elif plot_type=='grouped' and group_col:
    fig,ax=plt.subplots(figsize=(10,6))
    ct=pd.crosstab(df[group_col],df['Pheno_pred'],normalize='index')
    ct.plot(kind='barh',stacked=True,ax=ax,cmap='tab20',width=0.8,linewidth=1,edgecolor='black')
    ax.legend(bbox_to_anchor=(1.02,1),loc='upper left',fontsize=9,frameon=False)
    ax.set_title(f'Phenotype Distribution by {group_col}',fontsize=14,fontweight='bold');ax.set_xlabel('Proportion');sns.despine()
else:
    fig,ax=plt.subplots();ax.text(0.5,0.5,'Select a group column',ha='center',va='center')
plt.tight_layout();buf=io.BytesIO();fig.savefig(buf,format='png',dpi=150,bbox_inches='tight');plt.close(fig);buf.seek(0)
img_b64=base64.b64encode(buf.read()).decode('utf-8')
         `);
         document.getElementById('plot-img').src=`data:image/png;base64,${pyodide.globals.get('img_b64')}`;
         document.getElementById('plot-container').classList.add('visible');
         document.getElementById('download-btn').style.display='inline-block';log('Plot generated','success');
       }catch(e){log(`Error: ${e.message}`,'error');}
       btn.disabled=false;btn.textContent='Generate Plot';
     }

     function downloadMetaPlot(){const a=document.createElement('a');a.href=document.getElementById('plot-img').src;a.download='pangea_metaplot.png';a.click();}
   </script>

.. seealso::

   :doc:`02_vignette_meta_annotation`
      Step-by-step tutorial for cell and metadata annotation with PANGEApy.
