If I'm following your code, you have:

A form setup on the UI thread. Followed by...

A long running background thread. Followed by...

A form finalization on the UI thread.

You're looking to update an elapsed stopwatch throughout.

One way to accomplish this is with an elapsed update task that is not awaited, coupled with a background task that is awaited.

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
                    BeginInvoke(() =>labelElapsed.Text = $@"{stopwatch.Elapsed:hh\:mm\:ss\.f}");
                }
            }, cts.Token);
            bool Analyze = cboAnalyzePriceList.SelectedIndex == 0;
            bool Preview = cboPreviewOrSave.SelectedIndex == 0;
            // On UI Thread
            SetupProcessForm();
            // On Background thread
            var resultPAB = await ProcessActionButton(Analyze, Preview, cts.Token);
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