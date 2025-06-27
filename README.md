using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Text.RegularExpressions;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Input;
using System.Windows.Threading;

//Reference
/////OpenAI.(2022).ChatCPT(Novermber15 version)[Large language model].http://www.openai.com //Sosa,j and Stewart,B(2023).Blackbox ai(May version)[Large language mode].http://www.useblack.io/. //Microsoft.(2023) //orvalds, L., 2023. Linux Kernel Source Code. [source code] Available at: https://github.com/torvalds/linux [Accessed 14 May 2025]. //Doe, J., 2023. How to Build a Chatbot in C#. [online] CodeAcademy Blog. Available at: https://blog.codeacademy.com/chatbot-csharp [Accessed 14 May 2025].
/// https://github.com/ona-lenna/NewRepo3

namespace CybersecurityChatbotWPF
{
    public partial class MainWindow : Window
    {
        // Task class
        public class TaskItem
        {
            public string Title { get; set; }
            public string Description { get; set; }
            public DateTime? ReminderDate { get; set; }
            public bool IsCompleted { get; set; } = false;

            public string ReminderString => ReminderDate?.ToString("g") ?? "";
        }

        // Quiz question class
        private class QuizQuestion
        {
            public string Question { get; set; }
            public List<string> Options { get; set; }
            public int CorrectOptionIndex { get; set; }
            public string Explanation { get; set; }
        }

        private List<TaskItem> Tasks = new();
        private List<QuizQuestion> quizQuestions = new();
        private int currentQuizIndex = 0;
        private int quizScore = 0;
        private bool quizAnswered = false;
        private Random rnd = new();

        // Chatbot memory
        private string userName = "";
        private string favoriteTopic = "";

        // Activity log builder
        private StringBuilder activityLog = new();

        // FAQ dictionary
        private readonly Dictionary<string, string> faqResponses = new(StringComparer.OrdinalIgnoreCase)
        {
            { "what is phishing", "Phishing is a social engineering attack where attackers trick you into revealing sensitive info by pretending to be trustworthy." },
            { "how to enable 2fa", "To enable 2FA, activate two-factor authentication via your account security settings." },
            { "what is 2fa", "2FA means Two-Factor Authentication, adding an extra verification step beyond passwords." },
            { "how to create strong password", "Use uppercase, lowercase, numbers, symbols; avoid common words; at least 12 characters." },
            { "what is malware", "Malware is software designed to harm or exploit your system." }
        };

        // Cybersecurity tips dictionary
        private readonly Dictionary<string, List<string>> cybersecurityTips = new(StringComparer.OrdinalIgnoreCase)
        {
            { "password", new List<string>
                {
                    "A strong password is 12+ chars, mix of cases, numbers & symbols.",
                    "Use password managers to generate/store complex passwords.",
                    "Never reuse passwords across multiple accounts."
                }
            },
            { "scam", new List<string>
                {
                    "Be skeptical of unsolicited requests for personal info.",
                    "Never send money or info to unknown contacts.",
                    "Report suspicious scams to authorities or providers."
                }
            },
            { "privacy", new List<string>
                {
                    "Review app permissions regularly to limit data shared.",
                    "Use encrypted tools to protect your messages.",
                    "Avoid sharing sensitive info on social media."
                }
            }
        };

        // Sentiment responses dictionary
        private readonly Dictionary<string, List<string>> sentimentResponses = new(StringComparer.OrdinalIgnoreCase)
        {
            { "worried", new List<string>
                {
                    "Those concerns are justified. I can help you stay safe online.",
                    "It's okay to be worried, I'm here to help you with cybersecurity tips."
                }
            },
            { "curious", new List<string>
                {
                    "Great curiosity! What would you like to learn about cybersecurity?",
                    "Ask me anything you want to know about online safety!"
                }
            },
            { "frustrated", new List<string>
                {
                    "I understand it can be frustrating. Let's tackle this step by step.",
                    "Don't worry, I'll guide you through cybersecurity concepts easily."
                }
            }
        };

        public MainWindow()
        {
            InitializeComponent();

            InitializeQuiz();

            Loaded += (s, e) =>
            {
                DisplayAsciiArt();
                AppendToChat("Chatbot: Hello! I am your Cybersecurity Assistant. Ask me anything or manage your tasks.");
                AppendToActivityLog("Session started.");
            };

            // Reminder timer every 1 minute
            DispatcherTimer reminderTimer = new()
            {
                Interval = TimeSpan.FromMinutes(1)
            };
            reminderTimer.Tick += ReminderTimer_Tick;
            reminderTimer.Start();
        }

        #region Chatbot Core

        private void AppendToChat(string text)
        {
            Dispatcher.Invoke(() =>
            {
                ChatTextBlock.Text += text + Environment.NewLine + Environment.NewLine;
                if (ChatTextBlock.Parent is ScrollViewer scroller)
                    scroller.ScrollToEnd();
            });
        }

        private void AppendToActivityLog(string text)
        {
            var entry = $"[{DateTime.Now:HH:mm:ss}] {text}";
            activityLog.AppendLine(entry);
            Dispatcher.Invoke(() =>
            {
                ActivityLogTextBlock.Text = activityLog.ToString();
            });
        }

        private void DisplayAsciiArt()
        {
            string asciiArt = @"
  ____                 _                _                 
 |  _ \ ___  __ _  ___| | ___   _ _ __ | |_ ___ _ __ ___  
 | |_) / _ \/ _` |/ __| |/ / | | | '_ \| __/ _ \ '__/ __| 
 |  _ <  __/ (_| | (__|   <| |_| | |_) | ||  __/ |  \__ \ 
 |_| \_\___|\__,_|\___|_|\_\\__,_| .__/ \__\___|_|  |___/ 
                                 |_|                      
";
            AppendToChat(asciiArt);
        }

        private void SendButton_Click(object sender, RoutedEventArgs e)
        {
            var userText = UserInputTextBox.Text.Trim();
            if (string.IsNullOrEmpty(userText)) return;

            AppendToChat($"You: {userText}");
            AppendToActivityLog($"User: {userText}");
            UserInputTextBox.Clear();

            Task.Run(() => ProcessUserInput(userText));
        }

        private void UserInputTextBox_KeyDown(object sender, KeyEventArgs e)
        {
            if (e.Key == Key.Enter)
            {
                SendButton_Click(this, new RoutedEventArgs());
                e.Handled = true;
            }
        }

        private void ProcessUserInput(string input)
        {
            string text = input.ToLowerInvariant();

            // Greetings
            if (Regex.IsMatch(text, @"\b(hi|hello|hey|greetings)\b"))
            {
                RespondWithRandom(new[]
                {
                    "Hello! How can I assist you today?",
                    "Hi there! Ready to learn about cybersecurity?",
                    "Hey! What cybersecurity topic interests you?"
                });
                return;
            }

            // Memory: remember user name
            if (text.StartsWith("my name is "))
            {
                userName = input.Substring(11).Trim();
                AppendToChat($"Chatbot: Nice to meet you, {userName}!");
                AppendToActivityLog($"User set name: {userName}");
                return;
            }

            // Memory: favorite topic
            if (text.StartsWith("my favorite topic is "))
            {
                favoriteTopic = input.Substring(21).Trim();
                AppendToChat($"Chatbot: Got it! I'll remember that your favorite topic is {favoriteTopic}.");
                AppendToActivityLog($"User set favorite topic: {favoriteTopic}");
                return;
            }

            if (text.Contains("what is my favorite topic") || text.Contains("do you remember my favorite topic"))
            {
                if (!string.IsNullOrEmpty(favoriteTopic))
                    AppendToChat($"Chatbot: Your favorite cybersecurity topic is {favoriteTopic}.");
                else
                    AppendToChat("Chatbot: I don't know your favorite topic yet. You can tell me by saying, 'My favorite topic is phishing', for example.");
                return;
            }

            // Sentiment detection
            foreach (var sentiment in sentimentResponses.Keys)
            {
                if (text.Contains(sentiment))
                {
                    var response = GetRandomResponse(sentimentResponses[sentiment]);
                    AppendToChat("Chatbot: " + response);
                    AppendToActivityLog($"Sentiment detected: {sentiment}");
                    return;
                }
            }

            // Help command
            if (text.Contains("help"))
            {
                AppendToChat("Chatbot: You can ask me about phishing, 2FA, passwords, or say 'start quiz' to test your knowledge. You can also manage tasks.");
                return;
            }

            // Start quiz
            if (text.Contains("start quiz"))
            {
                Dispatcher.Invoke(() =>
                {
                    currentQuizIndex = 0;
                    quizScore = 0;
                    LoadQuizQuestion(currentQuizIndex);
                });
                AppendToChat("Chatbot: Starting quiz! Check the Quiz tab.");
                AppendToActivityLog("User started quiz.");
                return;
            }

            // Tasks info
            if (text.Contains("tasks") || text.Contains("task"))
            {
                AppendToChat("Chatbot: You can manage your cybersecurity tasks in the Tasks tab. Add tasks with reminders and mark them as complete.");
                return;
            }

            // FAQs
            foreach (var faqKey in faqResponses.Keys)
            {
                if (text.Contains(faqKey))
                {
                    AppendToChat("Chatbot: " + faqResponses[faqKey]);
                    AppendToActivityLog($"Answered FAQ: {faqKey}");
                    return;
                }
            }

            // Cybersecurity tips with random responses
            foreach (var tipKey in cybersecurityTips.Keys)
            {
                if (text.Contains(tipKey))
                {
                    var tip = GetRandomResponse(cybersecurityTips[tipKey]);
                    AppendToChat($"Chatbot tip on {tipKey}: {tip}");
                    AppendToActivityLog($"Provided tip on {tipKey}");
                    return;
                }
            }

            // Default fallback
            AppendToChat("Chatbot: I'm not sure I understand. Can you try rephrasing?");
            AppendToActivityLog("Fallback response given.");
        }

        private void RespondWithRandom(string[] responses)
        {
            var response = responses[rnd.Next(responses.Length)];
            AppendToChat("Chatbot: " + response);
            AppendToActivityLog($"Chatbot responded: {response}");
        }

        private string GetRandomResponse(List<string> responses)
        {
            if (responses == null || responses.Count == 0)
                return "Sorry, I don't have a response for that.";
            return responses[rnd.Next(responses.Count)];
        }

        #endregion

        #region Task Management

        private void AddTaskButton_Click(object sender, RoutedEventArgs e)
        {
            string title = TaskTitleTextBox.Text.Trim();
            string desc = TaskDescTextBox.Text.Trim();

            if (string.IsNullOrEmpty(title))
            {
                MessageBox.Show("Please enter a task title.", "Validation Error", MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }

            DateTime? reminderDate = null;
            if (TaskReminderDatePicker.SelectedDate.HasValue)
            {
                if (!string.IsNullOrEmpty(TaskReminderTimeTextBox.Text))
                {
                    if (TimeSpan.TryParse(TaskReminderTimeTextBox.Text, out var time))
                    {
                        reminderDate = TaskReminderDatePicker.SelectedDate.Value.Date + time;
                    }
                    else
                    {
                        MessageBox.Show("Invalid time format. Use HH:mm.", "Validation Error", MessageBoxButton.OK, MessageBoxImage.Warning);
                        return;
                    }
                }
                else
                {
                    reminderDate = TaskReminderDatePicker.SelectedDate.Value.Date;
                }
            }

            var task = new TaskItem { Title = title, Description = desc, ReminderDate = reminderDate };
            Tasks.Add(task);
            RefreshTasksList();

            TaskTitleTextBox.Clear();
            TaskDescTextBox.Clear();
            TaskReminderDatePicker.SelectedDate = null;
            TaskReminderTimeTextBox.Clear();

            AppendToActivityLog($"Task added: {title}");
        }

        private void RefreshTasksList()
        {
            TasksListView.ItemsSource = null;
            TasksListView.ItemsSource = Tasks.OrderBy(t => t.IsCompleted).ThenBy(t => t.ReminderDate ?? DateTime.MaxValue).ToList();
        }

        private void DeleteTaskButton_Click(object sender, RoutedEventArgs e)
        {
            if (TasksListView.SelectedItem is TaskItem selected)
            {
                Tasks.Remove(selected);
                RefreshTasksList();
                AppendToActivityLog($"Task deleted: {selected.Title}");
            }
            else
            {
                MessageBox.Show("Select a task to delete.", "Delete Task", MessageBoxButton.OK, MessageBoxImage.Information);
            }
        }

        private void CompleteTaskButton_Click(object sender, RoutedEventArgs e)
        {
            if (TasksListView.SelectedItem is TaskItem selected)
            {
                selected.IsCompleted = true;
                RefreshTasksList();
                AppendToActivityLog($"Task completed: {selected.Title}");
            }
            else
            {
                MessageBox.Show("Select a task to mark complete.", "Complete Task", MessageBoxButton.OK, MessageBoxImage.Information);
            }
        }

        private void TaskCompleted_Checked(object sender, RoutedEventArgs e)
        {
            if (sender is CheckBox cb && cb.DataContext is TaskItem task)
            {
                task.IsCompleted = true;
                AppendToActivityLog($"Task completed: {task.Title}");
                RefreshTasksList();
            }
        }

        private void TaskCompleted_Unchecked(object sender, RoutedEventArgs e)
        {
            if (sender is CheckBox cb && cb.DataContext is TaskItem task)
            {
                task.IsCompleted = false;
                AppendToActivityLog($"Task marked incomplete: {task.Title}");
                RefreshTasksList();
            }
        }

        private void ReminderTimer_Tick(object sender, EventArgs e)
        {
            var now = DateTime.Now;
            var dueTasks = Tasks.Where(t => t.ReminderDate.HasValue && !t.IsCompleted
                                           && t.ReminderDate.Value <= now
                                           && t.ReminderDate.Value > now.AddMinutes(-1)).ToList();

            foreach (var task in dueTasks)
            {
                Dispatcher.Invoke(() =>
                {
                    MessageBox.Show($"Reminder: Task '{task.Title}' is due now!", "Task Reminder", MessageBoxButton.OK, MessageBoxImage.Information);
                    AppendToActivityLog($"Reminder triggered for task: {task.Title}");
                });
            }
        }

        #endregion

        #region Quiz Management

        private void InitializeQuiz()
        {
            quizQuestions = new List<QuizQuestion>
            {
                new QuizQuestion
                {
                    Question = "What should you do if you receive an email asking for your password?",
                    Options = new List<string> { "Reply with password", "Delete the email", "Report the email as phishing", "Ignore it" },
                    CorrectOptionIndex = 2,
                    Explanation = "Correct! Reporting phishing emails helps prevent scams."
                },
                new QuizQuestion
                {
                    Question = "Phishing is:",
                    Options = new List<string> { "A type of malware", "A social engineering attack", "A firewall technique", "An encryption method" },
                    CorrectOptionIndex = 1,
                    Explanation = "Phishing is a social engineering attack to steal sensitive info."
                },
                new QuizQuestion
                {
                    Question = "Which one is a strong password?",
                    Options = new List<string> { "password123", "John1980", "S3cure#Pass!", "abcdefg" },
                    CorrectOptionIndex = 2,
                    Explanation = "Strong passwords use mix of cases, numbers, and symbols."
                },
                new QuizQuestion
                {
                    Question = "What does 2FA stand for?",
                    Options = new List<string> { "Two-Factor Authentication", "Two Firewall Access", "Third Factor Authorization", "Two File Access" },
                    CorrectOptionIndex = 0,
                    Explanation = "2FA means Two-Factor Authentication - extra security step."
                },
                new QuizQuestion
                {
                    Question = "True or False: You should share your password with close friends.",
                    Options = new List<string> { "True", "False" },
                    CorrectOptionIndex = 1,
                    Explanation = "False. Never share your passwords with anyone."
                },
                new QuizQuestion
                {
                    Question = "Safe browsing means:",
                    Options = new List<string> { "Clicking every link", "Only visiting trusted websites", "Downloading unknown files", "Ignoring browser warnings" },
                    CorrectOptionIndex = 1,
                    Explanation = "Safe browsing means only visiting trusted websites."
                },
                new QuizQuestion
                {
                    Question = "What is malware?",
                    Options = new List<string> { "Helpful software", "Software designed to harm", "Firewall", "Encryption method" },
                    CorrectOptionIndex = 1,
                    Explanation = "Malware is software designed to harm or exploit your system."
                },
                new QuizQuestion
                {
                    Question = "True or False: Updating software regularly helps security.",
                    Options = new List<string> { "True", "False" },
                    CorrectOptionIndex = 0,
                    Explanation = "True. Updates fix vulnerabilities and improve security."
                },
                new QuizQuestion
                {
                    Question = "Social engineering attacks rely on:",
                    Options = new List<string> { "Technical hacking", "Manipulating people", "Strong passwords", "Encryption" },
                    CorrectOptionIndex = 1,
                    Explanation = "They rely on manipulating people rather than technology."
                },
                new QuizQuestion
                {
                    Question = "A firewall is used to:",
                    Options = new List<string> { "Encrypt data", "Prevent unauthorized network access", "Steal passwords", "Delete files" },
                    CorrectOptionIndex = 1,
                    Explanation = "A firewall prevents unauthorized network access."
                }
            };

            Dispatcher.Invoke(() =>
            {
                currentQuizIndex = 0;
                quizScore = 0;
                LoadQuizQuestion(currentQuizIndex);
            });
        }

        private void LoadQuizQuestion(int index)
        {
            if (index >= quizQuestions.Count)
            {
                QuizQuestionTextBlock.Text = "Quiz Completed!";
                QuizOptionsPanel.Children.Clear();
                NextQuestionButton.IsEnabled = false;

                QuizResultTextBlock.Text = $"Your final score: {quizScore} out of {quizQuestions.Count}";

                string feedback = quizScore switch
                {
                    int n when n >= 8 => "Great job! You're a cybersecurity pro!",
                    int n when n >= 5 => "Good effort! Keep learning to stay safe online.",
                    _ => "Keep practicing! Cybersecurity is important for everyone."
                };
                AppendToChat($"Chatbot: Quiz finished. {feedback}");
                AppendToActivityLog($"Quiz completed. Score: {quizScore}/{quizQuestions.Count}");
                return;
            }

            var q = quizQuestions[index];
            QuizQuestionTextBlock.Text = $"Q{index + 1}: {q.Question}";
            QuizOptionsPanel.Children.Clear();
            QuizResultTextBlock.Text = "";
            NextQuestionButton.IsEnabled = false;
            quizAnswered = false;

            for (int i = 0; i < q.Options.Count; i++)
            {
                var rb = new RadioButton
                {
                    Content = q.Options[i],
                    Tag = i,
                    GroupName = "QuizOptions",
                    Margin = new Thickness(0, 0, 0, 8),
                    FontSize = 14
                };
                rb.Checked += QuizOption_Checked;
                QuizOptionsPanel.Children.Add(rb);
            }
        }

        private void QuizOption_Checked(object sender, RoutedEventArgs e)
        {
            if (quizAnswered) return;

            if (sender is RadioButton rb && rb.Tag is int selectedIndex)
            {
                quizAnswered = true;
                var q = quizQuestions[currentQuizIndex];

                if (selectedIndex == q.CorrectOptionIndex)
                {
                    quizScore++;
                    QuizResultTextBlock.Foreground = System.Windows.Media.Brushes.Green;
                    QuizResultTextBlock.Text = "Correct! " + q.Explanation;
                }
                else
                {
                    QuizResultTextBlock.Foreground = System.Windows.Media.Brushes.Red;
                    QuizResultTextBlock.Text = $"Incorrect. {q.Explanation}";
                }

                NextQuestionButton.IsEnabled = true;
                AppendToActivityLog($"Quiz Q{currentQuizIndex + 1} answered. Correct: {selectedIndex == q.CorrectOptionIndex}");
            }
        }

        private void NextQuestionButton_Click(object sender, RoutedEventArgs e)
        {
            currentQuizIndex++;
            LoadQuizQuestion(currentQuizIndex);
        }

        private void RestartQuizButton_Click(object sender, RoutedEventArgs e)
        {
            InitializeQuiz();
            AppendToActivityLog("Quiz restarted by user.");
        }

        #endregion
    }
}
