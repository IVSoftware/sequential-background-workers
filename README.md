If I'm following your code, you have:

1. A form setup on the UI thread. Followed by...
2. A long running background thread. Followed by...
3. A form finalization on the UI thread.

You're looking to update an elapsed stopwatch throughout.  
  
When I tried to reproduce this, for example by using the `Progress` class to report elapsed time, I was able to show #1 and #3 above not updating the elapsed time in spite of calls to `Report`.  
  
___
**Repro**

```
public partial class MainForm : Form, IProgress<TimeSpan>
{
    public MainForm()
    {
        InitializeComponent();
        btnAction.Click += btnAction_Click;
        _progress = new Progress<TimeSpan>();
        _progress.ProgressChanged += (sender, e) =>
        {
            labelElapsed.Text = $@"{stopwatch.Elapsed:hh\:mm\:ss\.f}";
            labelElapsed.Refresh();
        };
    }
    Progress<TimeSpan>? _progress;
    public void Report(TimeSpan value) => ((IProgress <TimeSpan>?)_progress)?.Report(value);

    ComboBox cboAnalyzePriceList = new(), cboPreviewOrSave = new(); // Dummies for MRE
    Stopwatch stopwatch = new ();
    private async void btnAction_Click(object? sender, EventArgs e)
    {
        try
        {
            btnAction.Enabled = false;
            labelElapsed.Visible = true;
            stopwatch.Restart();
            CancellationTokenSource cts = new();
            _ = Task.Run(() =>
            {
                while (!cts.Token.IsCancellationRequested)
                {
                    Report(stopwatch.Elapsed);
                    Thread.Sleep(100);
                }
            }, cts.Token);

            bool Analyze = cboAnalyzePriceList.SelectedIndex == 0;
            bool Preview = cboPreviewOrSave.SelectedIndex == 0;
            // On UI Thread
            SetupProcessForm();
            // On Background thread
            var resultPAB = await ProcessActionButton(Analyze, Preview, cts.Token);
            cts.Cancel();
            // On UI Thread
            FinalizeProcessForm();
            // (rowCnt, ImportId) = FinalizeProcessForm(result.rowCnt, result.FileMatch, result.srcAnalysis);
            PreviewOrSave();
            Text = "Main Form";
            MessageBox.Show($"{resultPAB.rowCnt} {resultPAB.fileMatch} {resultPAB.importId}");
        }
        catch(OperationCanceledException)
        { }
        finally
        {
            btnAction.Enabled = true;
            labelElapsed.Visible = false;
            stopwatch.Stop();
        }
    }

    private void PreviewOrSave() => Thread.Sleep(TimeSpan.FromSeconds(0.5));

    private async Task<(int rowCnt, bool fileMatch, string importId, object srcAnalysis)> ProcessActionButton(bool analyze, bool preview, CancellationToken token)
    {
        Text = "ImportValidateAnalyze";
        var result = await Task.Run(() =>
        {
            // "For clarification, ImportValidateAnalyze() is the
            // long-running process that I want to run on the separate thread."

            var importId = Guid.NewGuid().ToString().Trim(['{','}']);
            for (int i = 0; i < 10; i++)
            {
                Debug.Assert(InvokeRequired, "Ensure we're sleeping a non UI thread");
                Thread.Sleep(TimeSpan.FromSeconds(1)); // Simulated work
                if (token.IsCancellationRequested) return default;
            }
            // (rowCnt, FileMatch, ImportId, srcAnalysis) = mtdUpdateData.ImportValidateAnalyze(ImportId, PriceListFile, MappingName, Analyze, Preview);
            return (rowCnt: 100, FileMatch: true, ImportId: importId, srcAnalysis: new object());
        });
        return result;
    }

    // This attempts to simulate normal UI update
    // propagations without introducing any thread
    // waits or sleeps that might skew the results.
    // WHAT I AM CONFIRMING IS THAT REPORT FAILS TO UPDATE THE ELAPSED TIME.
    private void SetupProcessForm()
    {
        for (int i = 0; i < 1000000000; i++)
        {
            if ((i % 10000000 == 0))
            {
                Text = $"Form Setup {i / 10000000:D2}";
                Report(stopwatch.Elapsed);
                Refresh();
            }
        }
    }

    // This attempts to simulate normal UI update
    // propagations without introducing any thread
    // waits or sleeps that might skew the results.
    // WHAT I AM CONFIRMING IS THAT REPORT FAILS TO UPDATE THE ELAPSED TIME.
    private void FinalizeProcessForm()
    {
        for (int i = 0; i < 1000000000; i++)
        {
            if ((i % 10000000 == 0))
            {
                Text = $"Finalize Process Form {i / 10000000:D2}";
                Report(stopwatch.Elapsed);
                Refresh();
            }
        }
    }
}
```
___

**Workaround**

For this reason, I resorted to using `Invoke` with an elapsed update task that is not awaited, coupled with a background task that is awaited. Of all the experiments that I tried, this was the only one that produced reliable updates in this specific test scenario.

___

```
public partial class MainForm : Form
{
    public MainForm()
    {
        InitializeComponent();
        btnAction.Click += btnAction_Click;
    }

    ComboBox cboAnalyzePriceList = new(), cboPreviewOrSave = new(); // Dummies for MRE
    Stopwatch stopwatch = new ();
    private async void btnAction_Click(object? sender, EventArgs e)
    {
        try
        {
            btnAction.Enabled = false;
            labelElapsed.Visible = true;
            stopwatch.Restart();
            CancellationTokenSource cts = new();
            _ = Task.Run(() =>
            {
                while (!cts.Token.IsCancellationRequested)
                { 
                    Invoke(async () =>
                    {
                        labelElapsed.Text = $@"{stopwatch.Elapsed:hh\:mm\:ss\.f}";
                        await Task.Delay(TimeSpan.FromSeconds(0.1));
                    });
                }
            }, cts.Token);
            bool Analyze = cboAnalyzePriceList.SelectedIndex == 0;
            bool Preview = cboPreviewOrSave.SelectedIndex == 0;
            // On UI Thread
            SetupProcessForm();
            // On Background thread
            var resultPAB = await ProcessActionButton(Analyze, Preview, cts.Token);
            cts.Cancel();
            // On UI Thread
            FinalizeProcessForm();
            // (rowCnt, ImportId) = FinalizeProcessForm(result.rowCnt, result.FileMatch, result.srcAnalysis);
            PreviewOrSave();
            MessageBox.Show($"{resultPAB.rowCnt} {resultPAB.fileMatch} {resultPAB.importId}");
        }
        catch(OperationCanceledException)
        { }
        finally
        {
            btnAction.Enabled = true;
            labelElapsed.Visible = false;
            stopwatch.Stop();
        }
    }

    private void PreviewOrSave() => Thread.Sleep(TimeSpan.FromSeconds(0.5));

    private async Task<(int rowCnt, bool fileMatch, string importId, object srcAnalysis)> ProcessActionButton(bool analyze, bool preview, CancellationToken token)
    {
        Text = "ImportValidateAnalyze";
        var result = await Task.Run(() =>
        {
            // "For clarification, ImportValidateAnalyze() is the
            // long-running process that I want to run on the separate thread."

            var importId = Guid.NewGuid().ToString().Trim(['{','}']);
            for (int i = 0; i < 10; i++)
            {
                Thread.Sleep(TimeSpan.FromSeconds(1)); // Simulated work
                if (token.IsCancellationRequested) return default;
            }
            // (rowCnt, FileMatch, ImportId, srcAnalysis) = mtdUpdateData.ImportValidateAnalyze(ImportId, PriceListFile, MappingName, Analyze, Preview);
            return (rowCnt: 100, FileMatch: true, ImportId: importId, srcAnalysis: new object());
        });
        return result;
    }
    .
    .
    .
}
```