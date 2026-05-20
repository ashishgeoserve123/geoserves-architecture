# Hangfire Pause/Resume Bookmarklet

If the automatic JavaScript injection isn't working, you can use this bookmarklet as a backup solution.

## How to Create the Bookmarklet

1. Copy this entire JavaScript code (it's minified to work as a bookmarklet):

```javascript
javascript:(function(){if(document.querySelector('.hangfire-pause-btn')){console.log('Buttons already added');return;}const API_BASE='/api/report-engine/api/hangfirejob';function addButtons(){if(!window.location.href.includes('/recurring'))return;document.querySelectorAll('.btn-group').forEach(group=>{if(group.querySelector('.hangfire-pause-btn'))return;const row=group.closest('tr');if(!row)return;const checkbox=row.querySelector('input[type="checkbox"]');const jobId=checkbox&&checkbox.value?checkbox.value:'report-processing-job';const pauseBtn=document.createElement('button');pauseBtn.className='btn btn-sm btn-warning hangfire-pause-btn';pauseBtn.innerHTML='? Pause';pauseBtn.style.marginLeft='5px';pauseBtn.onclick=e=>{e.preventDefault();if(confirm(`Pause job "${jobId}"?`)){fetch(`${API_BASE}/pause/${jobId}`,{method:'POST'}).then(r=>r.json()).then(data=>{alert(data.success?`Job paused!`:`Error: ${data.message}`);if(data.success)location.reload();}).catch(err=>alert(`Error: ${err.message}`));}};const resumeBtn=document.createElement('button');resumeBtn.className='btn btn-sm btn-success hangfire-resume-btn';resumeBtn.innerHTML='? Resume';resumeBtn.style.marginLeft='5px';resumeBtn.onclick=e=>{e.preventDefault();if(confirm(`Resume job "${jobId}"?`)){fetch(`${API_BASE}/resume/${jobId}`,{method:'POST'}).then(r=>r.json()).then(data=>{alert(data.success?`Job resumed!`:`Error: ${data.message}`);if(data.success)location.reload();}).catch(err=>alert(`Error: ${err.message}`));}};group.appendChild(pauseBtn);group.appendChild(resumeBtn);console.log(`Added buttons for: ${jobId}`)});}addButtons();console.log('Hangfire Pause/Resume buttons added!');})();
```

2. **Create a new bookmark** in your browser
3. **Set the name** to: `Hangfire Pause/Resume`
4. **Set the URL** to the JavaScript code above
5. **Navigate to the Hangfire recurring jobs page**
6. **Click the bookmark** to add the pause/resume buttons

## Usage

1. Go to: `https://localhost:7162/api/report-engine/hangfire/recurring`
2. Click the "Hangfire Pause/Resume" bookmark
3. The pause and resume buttons will appear next to each job
4. Click the buttons to pause or resume jobs

This bookmarklet provides the same functionality as the automatic solution but requires manual activation each time you visit the page.