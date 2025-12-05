# Reference Wiki Thing

This is mostly just a collection of Org files and some logic to make it easy to reference things I'm always forgetting when it comes to development around Lisp things. The goal is to have a nice & navigable reference within Emacs (Doom right now, as thats what I'm using.)

## Installation

Expects this folder somewhere like - `~/references`. If you changed this, update the elisp accordingly.

## (Doom) Config

Add this to `~/.doom.d/config.el`:

``` emacs-lisp
(define-minor-mode reference-view-mode
  "Minor mode for viewing dev. reference files."
  :lighter " Ref"
  :keymap (let ((map (make-sparse-keymap)))
            (evil-define-key 'normal map (kbd "q") #'delete-window)
            map)
  (if reference-view-mode
      (view-mode)))

(defvar ref/references-dir "~/references/"
  "Directory containing reference org files.")

(defun ref/references ()
  "Open references index in a side window."
  (interactive)
  (let* ((index-file (expand-file-name "index.org" ref/references-dir))
         (buf (find-file-noselect index-file)))
    (with-current-buffer buf
      (reference-view-mode 1))
    (pop-to-buffer buf
                   '((display-buffer-in-side-window)
                     (side . right)
                     (window-width . 82)))))

(defun ref/org-open-ref-view-mode-same-window (orig-fn &rest args)
  "Force org links to open in same window when we're in reference-view-mode."
  (if (bound-and-true-p reference-view-mode)
      (let* ((context (org-element-context))
             (path (when (eq (org-element-type context) 'link)
                     (org-element-property :path context)))
             (full-path (when path
                          (expand-file-name path default-directory)))
             (existing-buf (when full-path (get-file-buffer full-path))))
        ;; Kill existing buffer if open elsewhere
        (when existing-buf
          (kill-buffer existing-buf)
          (print "Buffer: ~A~%" existing-buf))
        (let ((display-buffer-overriding-action '(display-buffer-same-window)))
          (apply orig-fn args)
          (when (and buffer-file-name
                     (string-prefix-p (expand-file-name ref/references-dir)
                                      buffer-file-name))
            (reference-view-mode 1))))
    (apply orig-fn args)))

(advice-add 'org-open-at-point :around #'ref/org-open-ref-view-mode-same-window)

(defun ref/references-search ()
  "Search all reference files by heading."
  (interactive)
  (let ((org-agenda-files (directory-files-recursively
                           ref/references-dir "\\.org$")))
    (consult-org-agenda)))

(defun ref/references-grep ()
  "Grep through all reference files."
  (interactive)
  (let ((default-directory ref/references-dir))
    (+default/search-cwd)))

(defun ref/references-find-file ()
  "Find a file in references directory."
  (interactive)
  (let ((default-directory ref/references-dir))
    (call-interactively #'find-file)))

;; Keybindings under SPC h (help prefix)
(map! :leader
      (:prefix ("h" . "help")
       :desc "References index"   "i"   #'ref/references
       :desc "References search"  "R"   #'ref/references-search
       :desc "References grep"    "C-r" #'ref/references-grep
       :desc "References file"    "M-r" #'ref/references-find-file))

;; Make org links open in same window (nicer for reference browsing)
(after! org
  (setq org-link-frame-setup
        '((file . find-file))))  ; instead of find-file-other-window
```

## Keybindings Summary

| Key           | Action                                |
|---------------+---------------------------------------|
| =SPC h r=     | Open references index (side window)   |
| =SPC h R=     | Search all references by heading      |
| =SPC h C-r=   | Grep through all reference files      |
| =SPC h M-r=   | Find file in references dir           |
| =SPC h l l=   | Jump directly to LOOP reference       |
| =SPC h l f=   | Jump directly to FORMAT reference     |
| =SPC h l c=   | Jump directly to CLOS reference       |
| =SPC h l i=   | Jump to Common Lisp index             |

## Navigation Within Org Files

| Key       | Action                                    |
|-----------+-------------------------------------------|
| =RET=     | Follow link (to other file or URL)        |
| =C-c &=   | Jump back after following a link          |
| =TAB=     | Fold/unfold current section               |
| =S-TAB=   | Cycle global visibility                   |
| =C-c C-j= | org-goto: jump to any heading             |
| =SPC s i= | imenu: fuzzy search headings              |
| =n= / =p= | Next/prev heading (at beginning of line)  |
| =q=       | Quit side window                          |

## Tips

### Create new reference files quickly

``` emacs-lisp
(defun ref/references-new (name)
  "Create a new reference file."
  (interactive "sNew reference name (e.g., cl/format): ")
  (let ((filepath (expand-file-name
                   (concat name ".org")
                   ref/references-dir)))
    (make-directory (file-name-directory filepath) t)
    (find-file filepath)
    (when (= (buffer-size) 0)
      (insert (format "#+TITLE: %s\n#+STARTUP: overview\n#+OPTIONS: toc:nil num:nil\n\n"
                      (file-name-base name)))
      (insert "[[file:../index.org][← Back]]\n\n* "))))
```

### Run Lisp examples with Sly

First, enable lisp in babel:

```elisp
(after! org
  (org-babel-do-load-languages
   'org-babel-load-languages
   '((lisp . t))))
```

## Todo
 - Look at using org-roam
