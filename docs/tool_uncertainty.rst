Uncertainty Explorer
====================

Visualize prediction uncertainty to identify missing or novel cell types.

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
     <p>Expected columns:</p>
     <div class="format">UMAP_1, UMAP_2, label, score_uncert, pred_uncert</div>
     <p style="margin-top:12px">label = cell type, score_uncert = uncertainty score, pred_uncert = True/False</p>
     <input type="file" id="file-input" accept=".csv" style="display:none" />
   </div>

   <div class="tool-controls" id="controls">
     <div class="control-group">
       <label for="plot-type">Plot Type</label>
       <select id="plot-type">
         <option value="umap_uncert">UMAP — Uncertain Cells</option>
         <option value="umap_score">UMAP — Uncertainty Score</option>
         <option value="bar_score">Bar — Mean Uncertainty by Label</option>
         <option value="bar_fraction">Bar — Fraction of Uncertain Cells</option>
       </select>
     </div>
     <div class="control-group">
       <label for="label-col">Label Column</label>
       <select id="label-col"></select>
     </div>
     <div class="control-group">
       <label>&nbsp;</label>
       <button class="primary" id="run-btn" onclick="generateUncertPlot()">Generate Plot</button>
     </div>
     <div class="control-group">
       <label>&nbsp;</label>
       <button class="download" id="download-btn" style="display:none" onclick="downloadUncertPlot()">Download PNG</button>
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
       const cols=(await pyodide.runPythonAsync(`
import pandas as pd
df=pd.read_csv('/data.csv')
skip={'UMAP_1','UMAP_2','score_uncert','pred_uncert','conf_score'}
[c for c in df.columns if c not in skip and not c.startswith('Unnamed')]
       `)).toJs();
       const ls=document.getElementById('label-col');ls.innerHTML='';
       for(const c of cols) ls.innerHTML+=`<option value="${c}">${c}</option>`;
       log(`Loaded: ${file.name} (${cols.length} label columns)`,'success');
       uploadArea.style.display='none';document.getElementById('controls').classList.add('visible');
       document.getElementById('loading').classList.remove('visible');stage('Ready!');
     }

     async function generateUncertPlot(){
       const btn=document.getElementById('run-btn');btn.disabled=true;btn.textContent='Generating...';stage('Generating plot...');
       const pt=document.getElementById('plot-type').value,lc=document.getElementById('label-col').value;
       try{
         await pyodide.runPythonAsync(`
import pandas as pd,numpy as np,matplotlib,io,base64
matplotlib.use('Agg')
import matplotlib.pyplot as plt;import seaborn as sns
df=pd.read_csv('/data.csv');plot_type='${pt}';label_col='${lc}'
if plot_type=='umap_uncert':
    fig,ax=plt.subplots(figsize=(10,8))
    if 'pred_uncert' in df.columns:
        df['pred_uncert']=df['pred_uncert'].astype(str)
        for val,color,lbl,sz,al in [('False','#cccccc','Confident',6,0.4),('True','#e74c3c','Uncertain',12,0.8)]:
            m=df['pred_uncert']==val;ax.scatter(df.loc[m,'UMAP_1'],df.loc[m,'UMAP_2'],c=color,s=sz,alpha=al,edgecolors='none',label=lbl)
        ax.legend(fontsize=11,frameon=False,markerscale=3)
    ax.set_xlabel('UMAP 1');ax.set_ylabel('UMAP 2')
    ax.set_title('Uncertain Cell Predictions',fontsize=14,fontweight='bold');ax.spines[['top','right']].set_visible(False)
elif plot_type=='umap_score':
    fig,ax=plt.subplots(figsize=(10,8))
    sc=ax.scatter(df['UMAP_1'],df['UMAP_2'],c=df['score_uncert'],cmap='OrRd',s=6,alpha=0.7,edgecolors='none')
    plt.colorbar(sc,ax=ax,label='Uncertainty Score',shrink=0.8)
    ax.set_xlabel('UMAP 1');ax.set_ylabel('UMAP 2')
    ax.set_title('Uncertainty Score Distribution',fontsize=14,fontweight='bold');ax.spines[['top','right']].set_visible(False)
elif plot_type=='bar_score':
    fig,ax=plt.subplots(figsize=(8,max(5,len(df[label_col].unique())*0.4)))
    order=df.groupby(label_col)['score_uncert'].mean().sort_values().index
    sns.barplot(data=df,y=label_col,x='score_uncert',order=order,errorbar='se',capsize=0.2,errwidth=1,errcolor='black',linewidth=1,edgecolor='black',ax=ax)
    ax.set_title('Mean Uncertainty Score by Cell Type',fontsize=14,fontweight='bold');ax.set_xlabel('Uncertainty Score');sns.despine()
elif plot_type=='bar_fraction':
    fig,ax=plt.subplots(figsize=(8,max(5,len(df[label_col].unique())*0.4)))
    if 'pred_uncert' in df.columns:
        df['pu']=df['pred_uncert'].astype(str).map({'True':True,'False':False})
        ct=pd.crosstab(df[label_col],df['pu'],normalize='index')
        if True in ct.columns: ct[True].sort_values().plot.barh(ax=ax,width=0.8,linewidth=1,edgecolor='black',color='salmon')
        ax.set_xlabel('Fraction of Uncertain Cells')
        ax.set_title('Uncertain Cell Fraction by Label',fontsize=14,fontweight='bold');sns.despine()
plt.tight_layout();buf=io.BytesIO();fig.savefig(buf,format='png',dpi=150,bbox_inches='tight');plt.close(fig);buf.seek(0)
img_b64=base64.b64encode(buf.read()).decode('utf-8')
         `);
         document.getElementById('plot-img').src=`data:image/png;base64,${pyodide.globals.get('img_b64')}`;
         document.getElementById('plot-container').classList.add('visible');
         document.getElementById('download-btn').style.display='inline-block';log('Plot generated','success');
       }catch(e){log(`Error: ${e.message}`,'error');}
       btn.disabled=false;btn.textContent='Generate Plot';
     }

     function downloadUncertPlot(){const a=document.createElement('a');a.href=document.getElementById('plot-img').src;a.download='pangea_uncertainty.png';a.click();}
   </script>

.. seealso::

   :doc:`03_vignette_identifying_missing_cells`
      Step-by-step tutorial for identifying missing cell types using uncertainty estimation.
